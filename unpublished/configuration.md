
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.08.02 - KivaKit components and settings](#configuration) <img src="https://state-of-the-art.org/graphics/star/star-16.png" srcset="https://state-of-the-art.org/graphics/star/star-16-2x.png 2x" style="vertical-align:top"/>  

<a name = "settings"></a>
<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

2021.08.02

### KivaKit components and settings &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears.svg" width="40"/>

The *kivakit-configuration* module provides three useful facilities for configuring and working with components:

1. Object lookup registry (*Registry*)
2. Settings registry (*Settings*)
3. Base component class (*BaseComponent*)

We will examine each of these in order. In practice, most applications will simply use *BaseComponent* (or *ComponentMixin*) instead of directly interacting with lookup and settings registries.

#### The object lookup registry

The class *Registry* in *com.telenav.kivakit.configuration.lookup* can be used to register and look up ordinary Java objects. For a detailed examination of the design pattern used by *Registry* and its advantages, see <a href="#service-locator">Why KivaKit provides service locator instead of dependency injection</a>. The preferred way to use *Registry* to register an object is:

    Spaceship spaceship = [...]
    
    Registry.of(this).register(spaceship);

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="https://state-of-the-art.org/graphics/saucer/saucer-80.png" srcset="https://state-of-the-art.org/graphics/saucer/saucer-80-2x.png 2x" style="vertical-align:middle"/> 

Our *Spaceship* object can then be found with:

    var spaceship = Registry.of(this).lookup(Spaceship.class);

If there is more than one *Spaceship* in the registry, is becomes necessary to distinguish which one we want to register or lookup. We do this by passing a second parameter that identifies which instance we are referring to:

    enum ShipType { BATTLESHIP, ORBITER }
    
    Spaceship spaceship = [...]

    Registry.of(this).register(spaceship, ORBITER);
    
    [...]
    
    var orbiter = Registry.of(this).lookup(Spaceship.class, ORBITER);

#### The settings registry

The *Settings* class in *com.telenav.kivakit.configuration.settings*, and its subclasses *SettingsFolder* and *SettingsPackage* provide registries of user-defined settings objects. For example, this user-defined settings object might configure *Apache Pinot*:

    public class PinotSettings
    {
        @KivaKitPropertyConverter
        Port zookeeperPort;
    
        @KivaKitPropertyConverter
        String clusterName;
    
        public Connection connection()
        {
            return ConnectionFactory.fromZookeeper(zookeeperPort + "/" + clusterName);
        }
    }

The *PinotSettings* object here specifies a port for *Apache Zookeeper*, and the name of an *Apache Pinot* cluster. These settings can be loaded from a *.properties* file in a *Folder* or a *Package*. The *.properties* file for *PinoSettings* looks like this:

    class         = com.telenav.scout.safety.demo.PinotSettings
    zookeeperPort = localhost:2181
    clusterName   = PinotCluster

The *class* value specifies the settings object to instantiate. Once the object is created, the *zookeeperPort* and *clusterName* values are converted to objects using the property converter specified (or implied by the field's type signature) by *@KivaKitPropertyConverter*. Those objects are then assigned to the appropriate fields of *PinotSettings*. Note that the *@KivaKitPropertyConverter* annotation can be applied to setter methods as well.

Packages and folders of *.properties* files can be used to group the settings for a particular configuration of an application or server (the *Deployment* and *DeploymentSet* classes help to do this and will be the subject of a future article). For example:

    settings
       |── PinotSettings.properties
       ├── WebSettings.properties
       └── HdfsSettings.properties

To load all settings objects from a folder into the settings registry for *this* object, the *addAllFrom()* method of *Settings* can be used like this:

    Settings.of(this).addAllFrom(Folder.parse("settings"));

When multiple instances of the same settings class are required (for example, two *Apache Pinot* clusters each defined by PinotSettings objects), 
an *instance* property can be used to distinguish which instance is defined in each *.properties* file:

    PinotCluster1.properties:

        class         = com.telenav.scout.safety.demo.PinotSettings
        instance      = CLUSTER1
        zookeeperPort = localhost:2181
        clusterName   = PinotCluster

    PinotCluster2.properties:

        class         = com.telenav.scout.safety.demo.PinotSettings
        instance      = CLUSTER2
        zookeeperPort = localhost:2181
        clusterName   = PinotCluster

These configurations can then be found using an instance specifier as in:

    enum Cluster { CLUSTER1, CLUSTER2 }

    var cluster1 = Settings.of(this).lookup(PinotSettings.class, CLUSTER1);
    var cluster2 = Settings.of(this).lookup(PinotSettings.class, CLUSTER2);


<a name="#components"></a>
#### KivaKit components

Now that we have covered the mechanisms for registering and locating objects and settings, we can take a look at how KivaKit components make this easier. KivaKit components, including *Application* and *Server* extend *BaseComponent*, which provides convenience methods for messaging, and for accessing objects from the lookup and settings registries for the component. In the event that a class already extends another base class, the *ComponentMixin* interface can be used (see [How KivaKit adds mixins to Java](#mixins) for details).

Continuing with our Apache Pinot example from above, we can write a *Pinot* component which uses *PinoSettings* to get a connection to the specified Apache Pinot database cluster:

    public class Pinot extends BaseComponent
    {
        public List<ResultSet> query(String query, Object... arguments)
        {
            final var connection = require(PinotSettings.class).connection();
            
            [...]
        }
    }

Notice that *require(PinotSettings.class)* in *BaseComponent* returns a *PinotSettings* object which, as we saw earlier, has no getters or setters. Instead it provides a method, *connection()* for getting a *Connection* based on the settings. The result is a one liner that gets a connection to our Pinot cluster without breaking encapsulation:

    require(PinotSettings.class).connection()

Note also that *BaseComponent* extends *BaseRepeater* and so all KivaKit components inherit messaging functionality.

Finally, the *Application* and *Server* classes in the *kivakit-application* module extend *BaseComponent*, which means that all applications have convenient ways to perform messaging operations and to look up objects and settings. In fact, in our *Application*, we can register our settings like this:

    public MyPinotApplication extends Application
    {
        protected void onRun()
        {
            registerSettingsIn(Folder.parse("settings/development"));
            
            [...]
        }
    }
    
In this code, all *.properties* files in the settings folder *settings/development* will be loaded into the application's settings registry. Note that the names of *.properties* files do not have to correspond to the settings class that they instantiate (as above in PinotCluster1.properties and PinotCluster2.properties).

Once the settings are loaded, they can be queried by the *Pinot* component, as shown above. In our *ApachePinot* example, we've shown how KivaKit's BaseComponent class (or ComponentMixin interface) makes it easy to register and locate settings objects. In the end, only two lines were required to load all of the application's settings objects:

            registerSettingsIn(Folder.parse("settings/development"));
            
and then access a database connection defined by *PinotSettings* elsewhere in the code:
            
            require(PinotSettings.class).connection()

The object and settings registries, and base component class covered above are all available in *kivakit-configuration* in the [KivaKit](https://www.kivakit.org) project.

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-configuration</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

Questions? Comments? Tweet yours to @OpenKivaKit.
