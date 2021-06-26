
<img src="lexakai-background-narrow-512.png" srcset="lexakai-background-narrow-512-2x.png"/>

<br/>
<br/>

### Articles

[**2021.06.26 - Constructors are evil (and how to eliminate them)**](#construction)  
[**2021.06.25 - How to add mixins to Java**](#mixins)  
[**‌2021.06.23 - Why I prefer the service locator design pattern over dependency injection**](#service-locator)  

<br/>
<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "construction"></a>

2021.06.26

### Constructors are evil (and how to eliminate them)

Methods for creating and initializing objects vary some from language to language, but most object-oriented languages allocate an object, often with a keyword such as *new*, and then perform object initialization using specialized methods called [constructors](https://tinyurl.com/5686t2km). One of the problems with constructors is that an object is not fully initialized until the constructor returns, which means that *during construction* the object is in a semi-initialized and possibly inconsistent state. This problem can be partly addressed with special compile-time checking, but even with this in place, constructors can still cause surprising and hard-to-diagnose problems:

1. Invoking overridable methods during construction can result in the use of data by subclasses that is not yet properly initialized.
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

We can specify this transition by adding two new keywords to our language: *is* and *when*:

    public class Alien
    {
        is ready when brain != null && spaceship != null;
        
        private Brain brain;
        private Spaceship spaceship;
        
        [..]
    }

The *is-when* statement here specifies that when our *allocated* *Alien* has both a *brain* and a *spaceship*, it will transition to the state *ready*.

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
    
    var alien = Alien;
    alien.brain(QuantumComputer);
    alien.spaceship(ZimsCoolSpaceship);
    alien.attack();

With our *is-when* statement, if we forget to add a *Brain* to the *Alien* the compiler can give us an error reminding us that we can't call the *attack()* method until the *Alien* has both a *brain* and a *spaceship*:

    var alien = Alien;
    alien.spaceship(ZimsCoolSpaceship);
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

Your comments?

<br/>
<br/>

<script src="https://utteranc.es/client.js"
        repo="jonathanlocke/jonathanlocke.github.io"
        issue-term="url"
        theme="github-dark"
        crossorigin="anonymous"
        async>
</script>

<br/>
<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "mixins"></a>

2021.06.25

### How to add mixins to Java

[Traits](https://tinyurl.com/2n6bbnv3) are a language feature in Scala, Groovy, Kotlin and other languages which allow groups of methods to be added to objects in arbitrary combinations. Unlike interfaces, however, traits can contain method bodies, which allows traits to provide objects with new behaviors. With the addition of [default methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html), Java now provides some of the features of traits, as described by Emil Forslund in [Traits in Java 8: Semantic, DRY-compliant, Interface-first Code](https://dzone.com/articles/definition-of-the-trait-pattern-in-java).

When traits include state, they are referred to as *stateful traits* or *mixins*. It is also possible to implement mixins in Java:

    public interface Mixin
    {
        /**
         * @param type The type of state to associate with this mixin
         * @param factory Creates state to associate with this mixin
         * @return The state for this mixin
         */
        default <T> T state(Class<? extends Mixin> type, Factory<T> factory)
        {
            // Retrieve the state for this mixin that is attached to this object
            return MixinState.get(this, type, factory);
        }
    }

Here, the *state()* method allows any interface extending *Mixin* to retrieve associated state for itself. If the state is not yet attached to the mix-in, it is created with the given *Factory*. The *MixinState* class that is used in the body of the *state()* method stores and retrieves state with a key that combines the object and the *Class* of the mixin. This allows each object to have state for many mixins associated with it. Note that an object using mixin(s) does not need to implement the *hashCode()* / *equals()* contract. Also note that mixin state is stored by *MixinState* using a *ConcurrentHashMap* to avoid concurrency problems. Access to this map may become contentious if there are a large number of objects accessing mixin state at the same time.

To define a new *Mixin*, we can simply extend the *Mixin* interface and use the *state()* method in our interface's default methods to access our state, as needed:

    public interface AttributedMixin<Key, Value> extends Mixin
    {
        default Value attribute(Key key)
        {
            return map().get(key);
        }
    
        default Value attribute(Key key, Value value)
        {
            return map().put(key, value);
        }
    
        private HashMap<Key, Value> map()
        {
            return state(AttributedMixin.class, () -> new HashMap<>());
        }
    }

*AttributedMixin* can now be added to any object, and it will provide a keyed attribute attached to that object. For example, it can be used like this to add a name to any arbitrary class:

    public class AttributedMixinTest
    {
        static class A implements AttributedMixin<String, String> { }
    
        static class B implements AttributedMixin<String, String> { }
    
        @Test
        public void test()
        {
            final var a = new A();
            final var b = new B();
    
            a.attribute("name", "This is object A");
            b.attribute("name", "This is object B");
    
            ensureEqual("This is object A", a.attribute("name"));
            ensureEqual("This is object B", b.attribute("name"));
        }
    }

This is a unit test for *AttributedMixin* from *kivakit-kernel* which is a module in [KivaKit](https://www.kivakit.org). 

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "service-locator"></a>

2021.06.23

### Why I prefer the service locator design pattern over dependency injection

Martin Fowler does a nice job of describing the *service locator* (SL) design pattern and *dependency injection* (DI) in his article [Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html#ServiceLocatorVsDependencyInjection). While I agree overall with what this article has to say (and I think we *all* agree on the principle of decoupling), I have a couple of fine points to add to the discussion. 

DI is not *fully* equivalent to SL. Both can be used effectively to decouple dependencies, and both get the job done. However, DI violates a core tenet of object-oriented programming: *encapsulation*. Whether you are doing constructor injection, setter injection or field injection, you are ultimately *pushing* an *implementation detail* into an object, which is the very definition of breaking encapsulation (for a detailed discussion on this subject, see Alan Holub's excellent and provocative article [Why getter and setter methods are evil](https://www.infoworld.com/article/2073723/why-getter-and-setter-methods-are-evil.html)).

Consider the case of an *Alien* that requires a *Brain* implementation. If we ignore all the arcane details of any particular DI framework, the dependency injection code for our alien might look like this:

    interface Brain 
    {
        boolean readyToAttack();
    }
	
    class Alien
    {
        @Inject
	    private Brain thinker;
	    
	    public void attackPlanet()
	    {
	        if (thinker.readyToAttack())
	        {
	              // TODO
	        }
	    }
    }
	
The DI framework here knows how to inject a *Brain* implementation into our *Alien* object by looking at the type (and potentially the name) of the private *thinker* field. This is simple enough, and although it does break encapsulation, it accomplishes the loose coupling we require.

Now, let's look at how SL might solve the same problem:

    class Alien
    {
        public void attackPlanet()
        {
	        var thinker = Registry.of(this).lookup(Brain.class);
	        if (thinker.readyToAttack())
	        {
	            // TODO
	        }	          
        }
    }
    
The statement *Registry.of(this)* finds the right *Registry* object to use for our *Alien* object (this is normally a global registry, but it could vary in some circumstances). Then the *lookup()* method yields an implementation of the *Brain* interface for the *Alien* to use. 

These two approaches seem identical at first glance, but there *is* one subtle difference. When the *attackPlanet()* method returns in the DI example, the *thinker* field still holds a reference to the *Brain* service. In the SL implementation the thinker reference is a local variable and when *attackPlanet()* returns, the *Brain* service implementation is no longer referenced and can potentially be garbage collected. Because encapsulation isn't broken, the *Alien* object can use a *Brain* implementation only *when it needs it*. In fact, if *attackPlanet()* is never called, there will be no *Brain* lookup at all.

Why should we care about this? Well, aside from purely ideological differences, the SL approach makes it very easy for a registry implementation to manage services more dynamically and potentially more efficiently. For example, a sophisticated registry could pool instances of non-thread-safe services that are expensive to create and use some form of concurrency control to restrict access. A registry implementation could also hold weak or soft references to services, allowing rarely used but memory hungry services to be collected when they're not actually in use. It could create some services with a factory on-the-fly. It could even vary the implementation of an interface over time. In each case, the SL design pattern is more flexible because the scope of reference to a service is an implementation detail, and with the SL pattern the consumer of a service can hold a reference to it exactly as long as it needs it.

A full implementation of the SL design pattern is available in [KivaKit](https://www.kivakit.org).

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

<sub>Copyright &#169; 2021 Jonathan Locke.</sub>  

<a title="Real Time Web Analytics" href="http://clicky.com/101323427"><img alt="Clicky" src="https://static.getclicky.com/media/links/badge.gif" border="0" /></a>
<script>var clicky_site_ids = clicky_site_ids || []; clicky_site_ids.push(101323427);</script>
<script async src="http://static.getclicky.com/js"></script>
