2021.10.14

### KivaKit Microservices &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears-32.png" srcset="https://state-of-the-art.org/graphics/gears/gears-32-2x.png 2x" style="vertical-align:baseline"/>

KivaKit is designed to make coding microservices faster and easier. In this blog post, we will examine the [*kivakit-microservice*](https://github.com/Telenav/kivakit-extensions/tree/develop/kivakit-microservice) module. As of this date, this module is only available for early access via SNAPSHOT builds and by [building KivaKit](https://github.com/Telenav/kivakit/blob/develop/documentation/overview/setup.md). The final release of KivaKit 1.1 will include this module and should happen by the end of October, 2021.

#### What does it do?

The *kivakit-microservice* mini-framework makes it easy to implement REST-ful GET, POST and DELETE request handlers, and to *mount* those handlers on specific paths. It also provides transparent support for GRPC using the same request handlers. Most of the usual plumbing for a REST microservice is taken care of, including:

* Configuration and startup of Jetty web server
* Handling GET, POST and DELETE requests
* Serialization of JSON objects with Json
* Error handling with KivaKit [messaging](2021-07-07-broadcaster.md)
* Generating an OpenAPI specification
* Viewing the OpenAPI specification with Swagger 
* Starting an Apache Wicket web application

#### Microservices

The *DivisionMicroservice* class below is a *Microservice* that performs arithmetic division (in the slowest and most expensive way imaginable). The *Microservice* superclass provides automatic configuration and startup of Jetty server:

    public class DivisionMicroservice extends Microservice
    {
        public static void main(final String[] arguments)
        {
            new DivisionMicroservice().run(arguments);
        }
    
        @Override
        public MicroserviceMetadata metadata()
        {
            return new MicroserviceMetadata()
                    .withName("division-microservice")
                    .withDescription("Example microservice for division")
                    .withVersion(Version.parse("1.0"));
        }
    
        @Override
        public void onInitialize()
        {
            // Register components here 
        } 
            
        @Override
        public DivisionRestService onNewRestService()
        {
            return new DivisionRestService(this);
        }
    }

Here, the *main(String[] arguments)* method creates an instance of *DivisionMicroservice* and starts it running with a call to *run(String[])* (the same as with any KivaKit application). The *metadata()* method returns information about the service that is included in the REST OpenAPI specification (mounted on */open-api/swagger.json*). The *onNewRestService()* factory method creates a REST service for the microservice, and the *webApplication()* factory method optionally creates an Apache Wicket web application for configuring the service and viewing its status. Any initialization of the microservice must take place in the *onInitialize()* method. This is the best place to [register components](2021-06-23-service-locator.md) used throughout the application. 

When the *run(String[] arguments)* method is called, Jetty web server is started on the port specified by the *MicroserviceSettings* object loaded by the [*-deployment* switch](2021-09-07-deployment.md). The *-port* command line switch can be used to override this value. 

When the microservice starts, the following resources are available:

| Resource Path          | Description                           |
|------------------------|---------------------------------------|
| /                      | Apache Wicket web application         |
| /                      | KivaKit microservlet REST service     |
| /assets                | Static resources                      |
| /docs                  | Swagger OpenAPI documentation         |
| /open-api/assets       | OpenAPI resources (.yaml files)       |
| /open-api/swagger.json | OpenAPI specification                 |
| /swagger/webapp        | Swagger web application               |
| /swagger/webjar        | Swagger design resources              |

#### REST Services

A REST service is created by extending the *MicroserviceRestService* class:

    public class DivisionRestService extends MicroserviceRestService
    {
        public DivisionRestService(Microservice microservice)
        {
            super(microservice);
        }
        
        @Override
        public void onInitialize()
        {
            mount("divide", POST, DivisionRequest.class);
        }
    }

Request handlers must be mounted on specific paths inside the *onInitialize()* method (or an error is reported). If the mount path (in this case "divide") doesn't begin with a slash ("/"), the path "/api/[major-version].[minor-version]/" is prepended automatically. So, "divide" becomes "/api/1.0/divide" in the code above, where the version *1.0* comes from the metadata returned by *DivisionMicroservice*. The *same path* can be used to mount a single request handler for each HTTP method (GET, POST, DELETE). However, trying to mount two handlers for the same HTTP method on the same path will result in an error.

The *gsonFactory()* factory method (not shown above) can optionally provide a factory that creates configured *Gson* objects. The *Gson* factory should extend the class *MicroserviceGsonFactory*. KivaKit will use this factory when serializing and deserializing JSON objects.

For anyone interested in the gory details, the exact flow of control that occurs when a request is made to a KivaKit microservice, is detailed in the Javadoc for [*MicroserviceRestService*](https://github.com/Telenav/kivakit-extensions/blob/develop/kivakit-microservice/src/main/java/com/telenav/kivakit/microservice/protocols/rest/MicroserviceRestService.java).

#### Microservlets

*Microservlets* handle *MicroservletRequest*s by producing *MicroservletResponse*s. They are mounted on paths in the same way that request handlers are mounted. Although *Microservlet* is a public API, so far the only use case is in dispatching requests to request handlers. The *MicroservletRestService* class uses an anonymous *Microservlet* class to forward requests to *MicroservletRequestHandler*s. 
You can see this in *MicroserviceRestService* in the method *mount(String path, HttpMethod method, Class&lt;MicroservletRequest&gt; requestType)*.

#### Microservlet Request Handlers

Request handlers are mounted on a *MicroserviceRestService* with calls to *mount(String path, HttpMethod method, Class&lt;MicroserviceRequest&gt; requestType)*. 

Note that the *BaseMicroservletRequest* class implements both *MicroservletRequest* and *MicroservletRequestHandler*:

    public abstract class BaseMicroservletRequest extends BaseComponent implements
            MicroservletRequest,
            MicroservletRequestHandler

This is crucial in terms of object-oriented design and encapsulation: *the request object itself has the data and therefore knows how to produce a response using that data*.

> **KEY DESIGN PRINCIPLE**
> 
> [*Don't ask for the information you need to do the work; ask the object that has the information to do the work for you.*](https://www.infoworld.com/article/2073723/why-getter-and-setter-methods-are-evil.html)
 
By employing this principle we avoid getters and setters, and our abstraction does not "leak".

Below we see a request handler, *DivisionRequest*, that divides two numbers. The request contains a dividend and a divisor. Its *onRequest()* method produces the response, which is implemented by the nested class *DivisionResponse*. The response has access to the dividend and the divisor in the outer request class. So, it can create the response quotient in its constructor:

    public DivisionResponse()
    {
        this.quotient = dividend / divisor;
    }

The request and response can also perform self-validation, again following the key design principle above, by overriding the *Validatable.validator()* method, as seen below.

> Note that an OpenAPI specification is generated using information from *@OpenApi* annotations

    @OpenApiIncludeType(description = "Request for divisive action")
    public class DivisionRequest extends BaseMicroservletRequest
    {
        @OpenApiIncludeType(description = "Response to a divide request")
        public class DivisionResponse extends BaseMicroservletResponse
        {
            @Tag(1)
            @Expose
            @OpenApiIncludeMember(description = "The result of dividing",
                                  example = "42")
            int quotient;
    
            public DivisionResponse()
            {
                this.quotient = dividend / divisor;
            }
    
            public String toString()
            {
                return Integer.toString(quotient);
            }
        }
    
        @Tag(1)
        @Expose
        @OpenApiIncludeMember(description = "The number to be divided",
                              example = "84")
        private int dividend;
    
        @Tag(2)
        @Expose
        @OpenApiIncludeMember(description = "The number to divide the dividend by",
                              example = "2")
        private int divisor;
    
        public DivisionRequest(int dividend, int divisor)
        {
            this.dividend = dividend;
            this.divisor = divisor;
        }
    
        public DivisionRequest()
        {
        }
    
        @Override
        @OpenApiRequestHandler(summary = "Divides two numbers")
        public DivisionResponse onRequest()
        {
            return listenTo(new DivisionResponse());
        }
    
        @Override
        public Class<DivisionResponse> responseType()
        {
            return DivisionResponse.class;
        }
    
        @Override
        public Validator validator(ValidationType type)
        {
            return new BaseValidator()
            {
                @Override
                protected void onValidate()
                {
                    problemIf(divisor == 0, "Cannot divide by zero");
                }
            };
        }
    }

We can appreciate now how time-tested OO design principles have improved encapsulation, eliminated boilerplate and increased code readability.

#### Accessing KivaKit Microservices in Java

The *kivakit-microservice* module includes *MicroserviceClient*, which provides easy access to KivaKit microservices in Java. The client can be used like this:

    public class DivisionClient extends Application
    {
        public static void main(String[] arguments)
        {
            new DivisionClient().run(arguments);
        }
    
        @Override
        protected void onRun()
        {
            var client = listenTo(new MicroservletClient(
                new MicroserviceGsonFactory(), 
                Host.local().https(8086), 
                Version.parse("1.0"));
    
            var response = client.post("divide", 
                DivisionRequest.DivisionResponse.class, 
                new DivisionRequest(9, 3));
    
            Message.println(AsciiArt.box("response => $", response));
        }
    }

Here, we create a *MicroservletClient* to access the microservice we built above. We tell it to use the service on port 8086 of the local host. Then we POST a *DivisionRequest* to divide 9 by 3 using the client, and we read the response. The response shows that the quotient is 3:

    -------------------
    |  response => 3  |
    -------------------

#### Path and Query Parameters

A request handler does not access path and query parameters directly. Instead they are automatically turned into JSON objects. For example, a POST to this URL:

    http://localhost:8086/api/1.0/divide/dividend/9/divisor/3

does the exact same thing as the POST request in the *DivisionClient* code above. The *dividend/9/divisor/3* part of the path is turned into a JSON object like this:

    {
        "dividend": 9,
        "divisor": 3
    }    

The microservlet processes this JSON just as if it had been posted. This feature can come in handy when POST-ing "flat" request objects (objects with no nesting). Note that when path variables or query parameters are provided, the body of the request is ignored.

#### OpenAPI

The "/docs" root path on the server provides a generated OpenAPI specification via Swagger:

![](https://www.state-of-the-art.org/graphics/swagger/swagger.png)

The annotations available for OpenAPI are minimal, but effective for simple REST projects:

| Annotation | Purpose                                                                            |
|------------|------------------------------------------------------------------------------------|
| @OpenApiIncludeMember | Includes the annotated method or field in the specification |
| @OpenApiExcludeMember | Excludes the annotation method or field from the specification |
| @OpenApiIncludeMemberFromSuperType | Includes a member from the superclass or superinterface in the specification       |
| @OpenApiIncludeType | Includes the annotated type in the specification schemas                           |
| @OpenApiRequestHandler | Provides information about a request handling method (*onGet()*, *onPost()* or *onDelete()*) |

#### GRPC

Google's remote procedure call protocol (GRPC) is transparently supported by the *kivakit-microservice* mini-framework thanks to the [Protostuff](https://github.com/protostuff/protostuff) project. To enable the GRPC protocol (on a separate port specified by the *-grpc-port* switch), two changes are required. The first is to create a trivial subclass of *MicroserviceGrpcService*. The second is to instantiate this class in *onNewGrpcService()*. That's it. Done.


    public class DivisionGrpcService extends MicroserviceGrpcService
    {
        public DivisionGrpcService(Microservice microservice)
        {
            super(microservice);
        }
    }

    public class DivisionMicroservice extends Microservice
    {
        [...]
            
        @Override
        public DivisionGrpcService onNewGrpcService()
        {
            return new DivisionGrpcService(this);
        }
    }
    
> Note: The *@Tag* annotations in the DivisionRequest and DivisionResponse objects above tell [Protostuff](https://github.com/protostuff/protostuff) what order the fields should go into the protocol buffer. This is necessary to allow for schema evolution. See the Protostuff documentation for details.
     
#### Code

The code discussed above is a [working example](https://github.com/Telenav/kivakit-examples/tree/develop/kivakit-examples-microservice) in the *kivakit-examples* repository. It can be instructive to trace through the code in a debugger.

The KivaKit Microservice API is available for early access in the *develop* branch of the [*kivakit-microservice*](https://github.com/Telenav/kivakit-extensions/tree/develop/kivakit-microservice) module of the *kivakit-extensions* repository in [KivaKit](https://www.kivakit.org).

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-microservice</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="microservices"
  theme="github-dark"
  crossorigin="anonymous"
></script>
