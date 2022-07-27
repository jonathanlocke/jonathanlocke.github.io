2021.06.26

### Constructors are evil (and how we could eliminate them) &nbsp; <img src="https://state-of-the-art.org/graphics/stars/stars-32.png" srcset="https://state-of-the-art.org/graphics/stars/stars-32-2x.png 2x" style="vertical-align:baseline"/>

Methods for creating and initializing objects vary some from language to language, but most object-oriented languages allocate an object, often with a keyword such as *new*, and then perform object initialization using specialized methods called [*constructors*](https://tinyurl.com/5686t2km). One of the problems with constructors is that an object is not fully initialized until the constructor returns, which means that *during construction* the object is in a semi-initialized and possibly inconsistent state. This problem can be partly addressed with special compile-time checking, but even with this in place, constructors can still cause surprising and hard-to-diagnose problems:

1. Invoking overridable methods during construction can result in the use of data by subclasses that are not yet properly initialized.
2. Constructors can allow the object's *this* reference to "escape" by being assigned to some field that is accessible by another thread. This other thread can then potentially use that field to access the object before its initialization has completed.
3. [Class initialization in Java can cause deadlocks](https://www.farside.org.uk/201510/deadlocks_in_java_class_initialisation)
4. Access to the static field(s) of a base class in Java does not cause the initialization of any subclasses, even though those subclasses might initialize these field(s). This can result in [counterintuitive behavior](https://stackoverflow.com/questions/10698516/behavior-of-static-blocks-with-inheritance).

In addition, fields are implicitly initialized during construction and the order of initialization is lexical. This can result in code that doesn't compile because the value of one field depends on another field that isn't yet initialized.

How can we solve some of the problems that constructors create?

Let's begin by thinking about how objects are initialized using a simple state diagram, where constructor invocation is represented by the state *initializing*:

    allocated -> initializing -> ready

In order to get rid of the problems with partially initialized objects, we need to get rid of the *initializing* state and transition directly from the allocated state to the ready state:

    allocated -> ready

Further, this transition must be *atomic*. An object should either be *allocated* or *ready*, but not in any state in between. But we still need to initialize the object's internal state at some point. How can we do this without including an *initializing* state?

Let's try a thought-experiment where we associate an actual state machine with each object. Initially this state machine will be in the *allocated* state. When the object has been fully initialized (by calling methods on the object), it will transition to *ready*. 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/alien/alien.svg" width="55" style="vertical-align:middle"/> 

We can specify this transition by adding two new keywords to our language: *is* and *when*:

    public class Alien
    {
        is ready when brain != null && spaceship != null;
        
        private Brain brain;
        private Spaceship spaceship;
        
        [..]
    }

The *is-when* statement here specifies that when our *allocated* *Alien* has both a *brain* and a *spaceship* (aliens have plug-and-play brains), it will transition to the state *ready*. 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="https://state-of-the-art.org/graphics/saucer/saucer.svg" width="60" style="vertical-align:middle"/> 

Now we can simply allow methods to mutate the *brain* and *ship* fields to trigger this transition. But some methods cannot function in the *allocated* state. These methods need to be *gated* until our *Alien* reaches the *ready* state. We can do this by adding the keyword *can*:

    public class Alien
    {
        is ready when brain != null && ship != null;
        can attack() when ready;
    
        private Brain brain;
        private Spaceship ship;
        
        public void brain(Brain brain) { this.brain = brain; }
        public void spaceship(Spaceship spaceship) { this.spaceship = spaceship; }
    
        public void attack()
        {
            if (brain.readyToAttack() && spaceship.readyToLiftOff())
            {
                // TODO
            }
        }
    }

The method *attack()* can only be called now when the *Alien* is in state *ready*. And the *Alien* is only *ready* if it has a *brain* and a *spaceship*, which implies that both the *brain()* and *spaceship()* methods have been called with non-null values.

One nice thing about this design is that eliminating constructors in favor of a state machine not only fixes our issue with partial initialization, but it also eliminates the clutter of constructor overloads and the confusion of constructors with too many parameters. Instead, initialization is both safe and clear:
    
    var alien = new Alien();
    alien.brain(quantumComputer);
    alien.spaceship(spaceshipZim);
    alien.attack();

With our *is-when* statement, if we forget to add a *Brain* to the *Alien* the compiler can give us an error reminding us that we can't call the *attack()* method until the *Alien* has both a *brain* and a *spaceship*:

    var alien = new Alien();
    alien.spaceship(spaceshipZim);
    alien.attack();
    
    [...]
    
    Compile error: method attack() requires Alien to be 'ready'

The *is-when* declaration makes our *Alien* more robust and also serves as a form of documentation, making it clear what has to be done to reach a given state and which methods can be invoked in each state.

So, perhaps we have moved closer to a solution for the object construction problem, but maybe we could use this state machine idea to solve other domain-specific problems. What if users could declare their own states?

Suppose we have a *SubspaceRadio* object that can be connected to a *SubspaceNetwork*. It might have additional states:

    public class SubspaceRadio
    {
        is ready when network != null;
        can connect when ready;
        can transmit when connected;
        can receive when connected;
        can disconnect when connected;
 
        private SubspaceNetwork network;

        public void network(SubspaceNetwork network) { this.network = network; }
        
        public void connect() 
        {
            if (network.connect(this))
            {
                is connected; 
            }
        }
        
        public void disconnect()
        {
            if (network.disconnect(this))
            {
                is ready;
            }
        }
        
        public void transmit()
        {
            [...]
        }
        
        public void receive()
        {
            [...]
        }
    }

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="https://state-of-the-art.org/graphics/state-machine/state-machine.svg" width="200" style="vertical-align:middle"/>

In this example, our radio object is *ready* as soon as it has a *network*. It can then transition to the *connected* state via a call to *connect()*, at which point it can *transmit()* and *receive()*. If *disconnect()* is called, the radio transitions back to *ready*, at which point it would have to reconnect with *connect()* before it can *transmit()* or *receive()* again.

Because the transition to *connected* is dynamic and conditional on whether the *network.connect()* method can establish a connection, this state machine transition cannot be determined at compile time. Instead, the compiler has to associate some of parts of the state machine with the object at runtime.

Note also that we might want to deny access to methods in a given state, such as if the radio is shut down. We can add another keyword and a wildcard pattern to permit this:

    cannot * when shutdown;
    
    public void shutdown() 
    {
        [...]
        
        is shutdown;
    }

The object can no longer be used once it is *shutdown*.

<br/>

<img src="https://telenav.github.io/telenav-assets/images/separators/horizontal-line-512.png" srcset="https://telenav.github.io/telenav-assets/images/separators/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="construction"
  theme="github-dark"
  crossorigin="anonymous"
></script>
