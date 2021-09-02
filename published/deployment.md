2021.09.07

### KivaKit deployments &nbsp; <img style="vertical-align:baseline; margin-left: 8px;" src="https://state-of-the-art.org/graphics/server/server.svg" height="24"/>

KivaKit applications can be configured using the [settings api](components-and-settings.md), but there is an easier way to configure an application for deployment built right into *Application*. By default, *Application* looks for the switch *-deployment=[deployment-name]*. If the switch is present and deployment settings can be found, KivaKit will load all of the settings objects in the named deployment into the global settings store, where they can be accessed with *require()*. 

Deployments can be packaged into a shaded jar file, so that usage of the application is very simple for an operations team:

    java -jar my-application.jar -deployment=local

To discover what *packaged* deployments are available, KivaKit looks in the *deployments* package next to the application class. Each sub-package in the *deployments* package is then a deployment, where the name of the deployment is the name of the package. A deployment description is contained in a file in the package named *Deployment.metadata*. The deployment package also contains a set of one or more *.properties* files. Each settings file describes a settings object, as described in [components and settings](components-and-settings.md).

    └── MyApplication.java
    └── deployments
        ├── local
        │   ├── Deployment.metadata
        │   └── Database.properties
        ├── development
        │   ├── Deployment.metadata
        │   └── Database.properties
        ├── staging
        │   ├── Deployment.metadata
        │   └── Database.properties
        └── production
            ├── Deployment.metadata
            └── Database.properties

It is very convenient to package up settings information in a *.jar* file in this way. To support external configuration, the *KIVAKIT_DEPLOYMENTS_FOLDER* system property can also be used to specify a deployment folder to load settings from, as in:

    java -jar my-application.jar -DKIVAKIT_DEPLOYMENT_FOLDER=$HOME/my-application/deployments/local

If a *deployments* package or external folder is found containing deployments, the *-deployment* switch is *required*. Failing to select a deployment will produce a usage message similar to:

    ┏--------- COMMAND LINE ERROR(S) ---------------
    ┋     ○ Required switch -deployment not found
    ┗-----------------------------------------------
    
    KivaKit 1.1.0 (beryllium alpaca)
    
    Usage: MyApplication 1.1.0 <switches> <arguments>
    
    This is my application.
    
    Arguments:
    
      <none>
    
    Switches:
    
      Required:
    
      -deployment=Deployment (required) : The deployment configuration to run
    
        ○ local - Run on localhost
        ○ development - Run on development cluster
        ○ staging - Run on staging cluster
        ○ production - Run in production environment
      
      Optional:
    
      -port=Integer (optional) : The port to use

If no deployments are available, the *-deployment* switch is not added to the application command line parser and cannot be used.

#### Code

The deployment configuration code that we've discussed here is available in the *kivakit-configuration* module in [KivaKit](https://www.kivakit.org).

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-configuration</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="deployment"
  theme="github-dark"
  crossorigin="anonymous"
></script>

