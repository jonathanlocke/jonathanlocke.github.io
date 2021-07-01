
#### <img src="https://state-of-the-art.org/kivakit-32.png" srcset="https://state-of-the-art.org/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.20 - Signaling and waiting for concurrent state changes](#state-watcher)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "state-watcher"></a>

2021.07.20

### Signaling and waiting for concurrent state changes &nbsp; <img src="https://state-of-the-art.org/graphics/communicate/communicate-32.png" srcset="https://state-of-the-art.org/graphics/communicate/communicate-32-2x.png 2x" style="vertical-align:baseline"/>

Java's concurrency library (java.util.concurrent) provides a mutual-exclusion (mutex) *Lock* called *ReentrantLock*. This lock maintains a queue of threads that are waiting to *own* the lock, allowing access to a protected resource. A thread can be added to the lock's wait queue by calling *lock()*. When the *lock()* method returns, the thread will own the lock. Once the thread obtains the lock in this way, it can mutate any shared state protected by the lock, and then it can release its ownership by calling *unlock()*, allowing another thread to get its turn at owning the lock and accessing the shared state. Because the lock is reentrant, a thread can call *lock()* multiple times, and the lock will only be released to the next waiting thread when all nested calls to *lock()* have been undone with calls to *unlock()*. The flow of a reentrant thread using a lock looks like this:

    lock() 
        lock() 
            lock() 
            unlock()
        unlock()
    unlock()

KivaKit provides a simple extension to this functionality that reduces boilerplate calls to *lock()* and *unlock()*, and ensures that all lock calls are balanced by unlock calls:

    public class Lock extends ReentrantLock
    {
        /**
         * Runs the provided code while holding this lock.
         */
        public void whileLocked(final Runnable code)
        {
            lock();
            try
            {
                code.run();
            }
            finally
            {
                unlock();
            }
        }
    }

Use of this class looks like:

    private Lock lock = new Lock();
    
    [...]
    
    lock.whileLocked(() -> mutateSharedState());

In addition to mutual exclusion, *ReentrantLock* (and in fact, all Java *Lock* implementations) provides an easy way for one thread to wait for a signal from another thread. This behavior makes *ReentrantLock* a *condition lock*, as declared in Java's *Lock* interface:

    public interface Lock
    {
        void lock();
        void unlock();
        Condition newCondition();
    }

The *Condition* implementation returned by *newCondition* has methods for threads that own the lock to signal or wait on the condition (similar to Java monitors). A simplification of the *Condition* interface looks like this:

    public interface Condition
    {
        void await() throws InterruptedException;
        void signal();
    }

KivaKit uses condition locks to create *StateWatcher*, which provides a way to signal and wait for a *state*.  

For example:

    enum State
    {
        IDLE,     // Initial state where nothing is happening
        WAITING,  // Signal that the foreground thread is waiting
        RUNNING,  // Signal that the background thread is running
        DONE      // Signal that the background thread is done
    }
    
    private StateWatcher state = new StateWatcher(IDLE);
    
    [...]
    
    new Thread(() ->
    {
        state.waitFor(WAITING); 
        state.signal(RUNNING);

        doThings();
        
        state.signal(DONE);
        
    }).start();
    
    state.signal(WAITING);
    state.waitFor(DONE);

In this example, you might expect that this code has a race condition. It is okay if the thread starts up and reaches *waitFor(WAITING)* before the foreground thread reaches *signal(WAITING)*. But what if the foreground thread signals that it's *WAITING* and proceeds to wait for *DONE* before the background thread even starts? With Java monitors (or *Conditions*), the signal would be missed by the background thread. It would then hang forever waiting for a *WAITING* signal that will never come. The foreground thread would also hang waiting for a *DONE* signal that will never arrive. This is a classic deadlock scenario.

*StateWatcher* solves this issue by making signaling and waiting *stateful* operations. In our race condition case, the foreground thread calls *signal(WAITING)*, as before. But the signal isn't lost. Instead, the watcher records that it is in the *WAITING* state before proceeding to wait for *DONE*. If the background thread then finishes starting up and it calls *waitFor(WAITING)*, the current state retained by *StateWatcher* will still be *WAITING* and the call will return immediately rather than doing any waiting. Our deadlock is eliminated, and with a minimal amount of code. The state that *StateWatcher* keeps to allow this to happen is commonly known as a *condition variable*.

But how exactly does StateWatcher implement this magic? 

*StateWatcher* has a *State* value that can be updated, and a (KivaKit) *Lock* that it uses to protect this state. It also maintains a list of *Waiter*s, each of which has a *Condition* to wait on (created from the *Lock*) and a *Predicate* that it needs to be satisfied.

When the *waitFor(Predicate<State>)* method is called (if the watcher isn't already in the desired *State*), a new *Waiter* is created and assigned the *Predicate* as well as a new *Condition* created from the *Lock*. The *waitFor()* method then adds the *Waiter* to the wait list and *awaits()* future signaling of the condition.

When *signal(State)* is called, the current state is updated, and then each waiter is processed. If a waiter's predicate is satisfied by the new state, its condition object is signaled, causing the thread awaiting satisfaction of the predicate to be awakened.

Finally, *waitFor(State)* is simply implemented with a method reference as a predicate: waitFor(desiredState::equals).

A simplified version of *StateWatcher* is shown below. The full *StateWatcher* class is available in kivakit-kernel in the [KivaKit](https://www.kivakit.org) project.

    public class StateWatcher<State>
    {
        /**
         * A thread that is waiting for its predicate to be satisfied
         */
        private class Waiter
        {
            /** The predicate that must be satisfied */
            Predicate<State> predicate;
    
            /** The condition to signal and wait on */
            Condition condition;
        }
    
        /** The KivaKit re-entrant lock */
        final transient Lock lock = new Lock();
    
        /** The clients waiting for a predicate to be satisfied */
        private final List<Waiter> waiters = new ArrayList<>();
    
        /** The most recently reported state */
        private volatile State current;
    
        public StateWatcher(final State current)
        {
            this.current = current;
        }
    
        /**
         * Signals any waiters if the state they are waiting for has arrived
         */
        public void signal(final State state)
        {
            lock.whileLocked(() ->
            {
                // Update the current state,
                current = state;
    
                // go through the waiters
                for (final var watcher : waiters)
                {
                    // and if the reported value satisfies the watcher's predicate,
                    if (watcher.predicate.test(state))
                    {
                        // signal it to wake up.
                        watcher.condition.signal();
                    }
                }
            });
        }
    
        /**
         * Waits for the given boolean predicate to be satisfied based on changes * to the observed state value
         */
        public WakeState waitFor(final Predicate<State> predicate)
        {
            return lock.whileLocked(() ->
            {
                // If the predicate is already satisfied,
                if (predicate.test(current))
                {
                    // we're done.
                    return COMPLETED;
                }
    
                // otherwise, add ourselves as a waiter,
                final var waiter = new Waiter();
                waiter.predicate = predicate;
                waiter.condition = lock.newCondition();
                waiters.add(waiter);
    
                try
                {
                    // and go to sleep until our condition is satisfied.
                    if (waiter.condition.await())
                    {
                        return TIMED_OUT;
                    }
                    else
                    {
                        return COMPLETED;
                    }
                }
                catch (final InterruptedException e)
                {
                    return INTERRUPTED;
                }
            });
        }

        /**
         * Wait forever for the desired state
         */
        public WakeState waitFor(final State desired)
        {
            return waitFor(desired::equals);
        }
    }
