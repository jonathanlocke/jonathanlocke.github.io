
<!-- <img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" /> -->

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
