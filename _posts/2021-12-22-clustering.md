2021.12.22

### KivaKit Clustering &nbsp; <img src="https://www.state-of-the-art.org/graphics/graph/graph.svg" width="40"/>

KivaKit provides built-in support for clustering of microservices using [Apache Zookeeper](https://zookeeper.apache.org). It supplies a cluster model
that is updated as members join and leave the cluster, and an implementation of the [*SettingsStore*](2021-08-02-components-and-settings.md) 
interface that stores settings in Zookeeper.

#### Joining and Leaving a KivaKit Microservice Cluster

To use KivaKit in a cluster, Apache Zookeeper must be running according to the instructions. The default port for Zookeeper is 2181.

The source code for a clustered microservice should be organized like this:

    ├── deployments
    │   └── mycluster
    │       ├── ZookeeperConnection.properties
    │       └── MyMicroserviceSettings.properties
    └── MyMicroservice

The *ZookeeperConnection.properties* file here configures *ZookeeperConnection* as specified by [kivakit-configuration](2021-08-02-components-and-settings.md):

    class       = com.telenav.kivakit.settings.stores.zookeeper.ZookeeperConnection$Settings
    ports       = 127.0.0.1:2181
    timeout     = 5m
    create-mode = PERSISTENT

The *Microservice* subclass, *MyMicroservice*, is then parameterized on a class that holds information about cluster members.
Each cluster member will have its own instance of this object, describing that particular member. An instance of this object 
is created during initialization by the onNewMember() method:

    public class MyMicroservice extends Microservice<MyMicroserviceSettings>
    {
        [...]
        

        protected MicroserviceClusterMember<MyMicroserviceSettings> onNewMember()
        {
            return new MicroserviceClusterMember<>(require(MyMicroserviceSettings.class));
        }
    }

Here, the *MicroserviceClusterMember* model returned by this method references an instance of the *MyMicroserviceSettings*,
which is created and registered during initialization, typically by a settings file in the deployment like MyMicroserviceSettings:

    class    = myapp.MyMicroserviceSettings
    port     = 8081
    grpcPort = 8082
    server   = true

Once the *MicroserviceClusterMember* model object has been created, it is stored in Zookeeper. Other microservice instances 
in the cluster then receive a notification that a new member has joined through this *Microservice* method:

    protected void onJoin(MicroserviceClusterMember<MyMicroserviceSettings> member)
    {
        announce("Joined cluster: $", member.identifier());
    }

&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/footprints/footprints.svg" width="32"/>

When a member leaves the cluster, its model object will disappear from Zookeeper, and the remaining cluster members
will be notified with a call to:

    protected void onLeave(MicroserviceClusterMember<MyMicroserviceSettings> member)
    {
        announce("Left cluster: $", member.identifier());
    }
    
#### Cluster Elections

A cluster has an elected *leader* at any given time. Each time a member joins or leaves the cluster, an election is held
to determine which member should lead the cluster. The election takes place automatically and the elected member will have 
the first *MicroserviceClusterMember.identifier()* value alphabetically. This identifier is currently the DNS name of 
the host and the process number, but is only guaranteed to be unique and is subject to change in the future.

To determine if a cluster member is the elected leader:

    if (member.isLeader())
    {
        [...]
    }

#### Zookeeper Settings Stores &nbsp; <img src="https://www.state-of-the-art.org/graphics/disks/disks.svg" width="40"/>

Although some applications may require only the settings object provided by *MicroserviceClusterMember*, others
may need to store settings for other components in Zookeeper. This can be accomplished by registering a *ZookeeperSettingsStore*
instance in *Microservice.onInitialize()*:

    var store = listenTo(register(new ZookeeperSettingsStore(PERSISTENT)));

Settings can be loaded from this store with

    registerSettingsIn(store);

and settings can be saved to this store with:

    saveSettingsTo(store, settings);

Because settings are retrieved in KivaKit using a *pull* model, any changes to settings objects will be 
automatically available when the desired object is next retrieved with *require(Class)*.

A typical idiom is to read any existing settings from the store, and if the desired settings object is not present,
save a default value. Other members will then read that value using the same logic:

    registerSettingsIn(store);
    if (!hasSettings(MyMicroserviceSettings.class))
    {
       store.save(new MyMicroserviceSettings());
       registerSettingsIn(store);
    }

Manually modifying the GSON value in Zookeeper will have the identical effect to saving a value with *saveSettingsTo()*.

#### Code

The code discussed above is available on GitHub:

 - [kivakit-microservice](https://github.com/Telenav/kivakit-extensions/tree/master/kivakit-microservice)  
 - [kivakit-settings-stores-zookeeper](https://github.com/Telenav/kivakit-extensions/tree/master/kivakit-settings-stores/zookeeper)  

The KivaKit Microservice API is available on *Maven Central* at these coordinates:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-microservice</artifactId>
        <version>1.2.0</version>
    </dependency>
    
The KivaKit Zookeeper settings store API is at these coordinates:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-settings-stores-zookeeper</artifactId>
        <version>1.2.0</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="clustering"
  theme="github-dark"
  crossorigin="anonymous"
></script>
