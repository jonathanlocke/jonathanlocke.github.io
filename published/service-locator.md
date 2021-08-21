2021.06.23

### Why KivaKit uses service locator instead of DI &nbsp; <img src="https://state-of-the-art.org/graphics/link/link.svg" width="25" style="vertical-align:bottom"/>

Martin Fowler does a nice job of describing the *service locator* (SL) design pattern and *dependency injection* (DI) in his article [Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html#ServiceLocatorVsDependencyInjection). The basic distinction between these two patterns is that in DI, a container pushes interfaces into an object based on its configuration while in SL, the object reaches out to the container to ask for the interface. While I agree overall with what this article has to say (and I think we *all* agree on the principle of decoupling), I have a couple of fine points to add to the discussion.

DI is not *fully* equivalent to SL. Both can be used effectively to decouple dependencies, and both get the job done. However, DI violates a core tenet of object-oriented programming: *encapsulation*. Whether you are doing constructor injection, setter injection or field injection, you are ultimately *pushing* an *implementation detail* into an object, which is the very definition of breaking encapsulation (for a detailed discussion on this subject, see Alan Holub's excellent and provocative article [Why getter and setter methods are evil](https://www.infoworld.com/article/2073723/why-getter-and-setter-methods-are-evil.html)).

Consider the case of an *Alien* that requires a *QuantumDatabase* implementation to find out what planet to attack next. If we ignore all the arcane details of any particular DI framework, the dependency injection code for our alien might look like this:

    interface QuantumDatabase 
    {
        Planet queryPlanetToAttack(Alien alien);
    }
	
    class Alien
    {
        @Inject
	    private QuantumDatabase database;
	    
	    public void attackPlanet()
	    {
        	    var planet = database.queryPlanetToAttack(this);
        	    
        	    // TODO
	    }
    }
	
The DI framework here knows how to inject a *QuantumDatabase* implementation into our *Alien* object by looking at the type (and potentially the name) of the private *database* field. This is simple enough, and although it does break encapsulation, it accomplishes the loose coupling we require.

Now, let's look at how SL might solve the same problem:

    class Alien
    {
        public void attackPlanet()
        {
	        var planet = Registry.of(this)
	            .lookup(QuantumDatabase.class)
	            .queryPlanetToAttack(this);
	            
	        // TODO
        }
    }
    
The statement *Registry.of(this)* finds the right *Registry* object to use to find our *Alien* object (this is normally a global registry, but it could vary in some circumstances). Then the *lookup()* method yields an implementation of the *QuantumDatabase* interface for the *Alien* to use.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-lookup.svg" width="300"/>

These two approaches seem identical at first glance, but there *is* one subtle difference. When the *attackPlanet()* method returns in the DI example, the *database* field still holds a reference to the *QuantumDatabase* service. The alien and its database have the same lifecycle. However, in the SL implementation the *database* reference is a local and when *attackPlanet()* returns, the *QuantumDatabase* service implementation is no longer referenced and can potentially be garbage collected. Because encapsulation isn't broken, the *Alien* object can use a *QuantumDatabase* implementation only *when it needs it*. In fact, if *attackPlanet()* is never called, there will be no *QuantumDatabase* lookup at all (and potentially the *QuantumDatabase* won't be constructed either, saving on energy used by the particle accelerator).

Why should we care about this? Well, aside from purely ideological differences, the SL approach makes it very easy for a registry implementation to manage services more dynamically, and potentially more efficiently as well. For example, a sophisticated registry could pool instances of non-thread-safe services that are expensive to create, and use some form of concurrency control to restrict access. A registry implementation could also hold weak or soft references to services, allowing rarely used but memory hungry services to be collected when they're not actually in use. It could create some services with a factory on-the-fly. It could even vary the implementation of an interface over time. In each case, the SL design pattern is more flexible because the scope of reference to a service is an implementation detail, and with the SL pattern the consumer of a service can hold a reference to it exactly as long as it needs it.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

A full implementation of the SL design pattern is available in [KivaKit](https://www.kivakit.org).

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-kernel</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="service-locator"
  theme="github-dark"
  crossorigin="anonymous"
></script>

