
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.08.02 - KivaKit components and settings](#configuration) <img src="https://state-of-the-art.org/graphics/star/star-16.png" srcset="https://state-of-the-art.org/graphics/star/star-16-2x.png 2x" style="vertical-align:top"/>  

<a name = "settings"></a>
<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

2021.08.02

### KivaKit components and settings &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears.svg" width="40"/>

The *kivakit-configuration* module provides three useful facilities for configuring and working with components:

1. Object lookup registry (*Registry*)
2. Settings registry (*Settings*)
3. Base component class (*BaseComponent*)

We will examine each of these in order.

#### The object lookup registry

The class *Registry* in *com.telenav.kivakit.configuration.lookup* can be used to register and look up Java objects. For a detailed examination of the design pattern used by *Registry* and its advantages, see <a href="#service-locator">Why KivaKit provides service locator instead of dependency injection</a>. The preferred way to use *Registry* to register an object is:

    Spaceship spaceship = [...]
    
    Registry.of(this).register(spaceship);

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="https://state-of-the-art.org/graphics/saucer/saucer-80.png" srcset="https://state-of-the-art.org/graphics/saucer/saucer-80-2x.png 2x" style="vertical-align:middle"/> 

Our *Spaceship* object can then be found with:

    var spaceship = Registry.of(this).lookup(Spaceship.class);

If there is more than one *Spaceship* in the registry, is becomes necessary to distinguish which one we want to register or lookup. We do this by passing a second parameter that identifies which instance we are referring to:

    enum ShipType { BATTLESHIP, ORBITER }
    
    Spaceship spaceship = [...]

    Registry.of(this).register(spaceship, ShipType.ORBITER);
    
    [...]
    
    var orbiter = Registry.of(this).lookup(Spaceship.class, ShipType.ORBITER);

#### The settings registry

The *Settings* class in *com.telenav.kivakit.configuration.settings*, and its subclasses *SettingsFolder* and *SettingsPackage* provide access to configuration objects in the form of user-defined settings objects. For example, this settings object might be used to configure Apache Pinot:

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

The *PinotSettings* object here specifies a port for Apache Zookeeper and the name of an Apache Pinot cluster. These settings can be loaded from a *.properties* file in a *Folder* or a *Package*. The *.properties* file for *PinoSettings* looks like this:

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

#### KivaKit components

Now that we have covered the mechanisms for registering and locating objects and settings, we can forget about it for now (until some later date if it becomes relevant). All of the above functionality is more easily accessible by creating a KivaKit component. KivaKit components, including *Application*, extend *BaseComponent*, which provides convenience methods for accessing objects from the lookup registry and the settings registry for the component. 

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

Finally, the Application class in the *kivakit-application* module extends *BaseComponent*, which means that all applications have convenient ways to perform messaging operations and to look up objects and settings. In fact, in our *Application*, we might wish to register our settings like this:

    public MyPinotApplication extends Application
    {
        protected void onRun()
        {
            registerSettingsIn(Folder.parse("settings"));
            
            [...]
        }
    }
    
In this code, the settings folder containing *PinotSettings.properties* will be loaded into the application's settings registry so it can be queried by the *Pinot* component.

The code covered above is available in *kivakit-configuration* in the [KivaKit](https://www.kivakit.org) project.

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-configuration</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

Questions? Comments? Tweet yours to @OpenKivaKit.
