2021.10.01

### KivaKit Microservices &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears-32.png" srcset="https://state-of-the-art.org/graphics/gears/gears-32-2x.png 2x" style="vertical-align:baseline"/>

KivaKit is designed to make coding microservices faster and easier. In this blog post, we will examine the [*kivakit-microservice*](https://github.com/Telenav/kivakit-extensions/tree/develop/kivakit-microservice) module. As of this date, this module is only available for early access via SNAPSHOT builds and by [building KivaKit](https://github.com/Telenav/kivakit/blob/develop/documentation/overview/setup.md). The final release of KivaKit 1.1 will include this module and should happen by the end of October, 2021 or sooner.

#### What does it do?

The *kivakit-microservice* mini-framework makes it easy to implement REST-ful GET, POST and DELETE handlers, and to *mount* those handlers on specific paths. Most of the usual plumbing for a REST microservice is taken care of, including:

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
                    .withName("divide-microservice")
                    .withDescription("Example microservice for division")
                    .withVersion(Version.parse("1.0"));
        }
    
        @Override
        public void onInitialize()
        {
            // Register components here 
        } 
            
        public DivideRestApplication restApplication()
        {
            return new DivideRestApplication(this);
        }
    }

Here, the *main(String[] arguments)* method creates an instance of *DivisionMicroservice* and starts it running with a call to *run(String[])* (the same as with any KivaKit application). The *metadata()* method returns information about the service that is included in the REST OpenAPI specification (mounted on /open-api/swagger.json). The *restApplication()* factory method creates a REST application for the microservice, and the *webApplication()* factory method optionally creates an Apache Wicket web application for configuring the service and viewing its status. Any initialization of the microservice must take place in the *onInitialize()* method. This is the best place to [register components](2021-06-23-service-locator.md) used throughout the application. 

When the *run(String[] arguments)* method is called, Jetty web server is started on the port specified by the *MicroserviceSettings* object loaded by the [*-deployment* switch](2021-09-07-deployment.md). The *-port* command line switch can be used to override this value. 

When the microservice starts, the following resources are available:

| Resource Path          | Description                           |
|------------------------|---------------------------------------|
| /                      | Apache Wicket web application         |
| /                      | KivaKit microservlet REST application |
| /assets                | Static resources                      |
| /docs                  | Swagger OpenAPI documentation         |
| /open-api/assets       | OpenAPI resources (.yaml files)       |
| /open-api/swagger.json | OpenAPI specification                 |
| /swagger/webapp        | Swagger web application               |
| /swagger/webjar        | Swagger design resources              |

#### REST Applications

A REST application is created by extending the *MicroserviceRestApplication* class:

	public class DivideRestApplication extends MicroserviceRestApplication
	{
		public DivideRestApplication(Microservice microservice)
		{
			super(microservice);
		}
		
		@Override
		public void onInitialize()
		{
			mount("divide", DivideRequest.class);
		}
	}

Request handlers must be mounted on specific paths inside the *onInitialize()* method (or an error is reported). If the mount path (in this case "divide") doesn't begin with a slash ("/"), the path "/api/[major-version].[minor-version]/" is prepended automatically. So, "divide" becomes "/api/1.0/divide" in the code above, where the version *1.0* comes from the metadata returned by *DivideMicroservice*. The *same path* can be used to mount a single request handler for each HTTP method (GET, POST, DELETE). However, trying to mount two handlers for the same HTTP method on the same path will result in an error.

The *gsonFactory()* factory method (not shown above) can optionally provide a factory that creates configured *Gson* objects. The *Gson* factory should extend the class *MicroserviceGsonFactory*. KivaKit will use this factory when serializing and deserializing JSON objects.

For anyone interested in the gory details, the exact flow of control that occurs when a request is made to a KivaKit microservice, is detailed in the Javadoc for [*MicroserviceRestApplication*](https://github.com/Telenav/kivakit-extensions/blob/develop/kivakit-microservice/src/main/java/com/telenav/kivakit/microservice/rest/MicroserviceRestApplication.java).

#### Microservlets

*Microservlets* handle GET, POST and DELETE requests. They are mounted on paths in the same way that request handlers are mounted. But unlike a request handler, a microservlet can handle any or all HTTP request methods at the same time. Request handlers are more flexible and generally more useful than microservlets, so this information is mainly here for the sake of completeness. The key use case (the only one so far) for microservlets is that they are used to implement request handlers. You can see the internal microservlet for this in *MicroserviceRestApplication* in the method *mount(String path, Class&lt;MicroservletRequest&gt; requestType)*.

#### Request Handlers

Request handlers are mounted on a *MicroserviceRestApplication* with calls to *mount(String path, Class&lt;MicroserviceRequest&gt; requestType)*. They come in three flavors, each of which is a subclass of *MicroserviceRequest*:

* MicroservletGetRequest
* MicroservletPostRequest
* MicroservletDeleteRequest

Below, we see a POST request handler, *DivideRequest*, that divides two numbers. The response is formulated by the nested class *DivideResponse*. An OpenAPI specification is generated using information from the *@OpenApi* annotations. Finally, the request performs self-validation by implementing the *Validatable* interface required by *MicroservletPostRequest*:

    @OpenApiIncludeType(description = "Request for divisive action")
    public class DivideRequest extends MicroservletPostRequest
    {
        @OpenApiIncludeType(description = "Response to a divide request")
        public class DivideResponse extends MicroservletResponse
        {
            @Expose
            @OpenApiIncludeMember(description = "The result of dividing",
                                  example = "42")
            int quotient;
    
            public DivideResponse()
            {
                this.quotient = dividend / divisor;
            }
    
            public String toString()
            {
                return Integer.toString(quotient);
            }
        }
    
        @Expose
        @OpenApiIncludeMember(description = "The number to be divided",
                              example = "84")
        private int dividend;
    
        @Expose
        @OpenApiIncludeMember(description = "The number to divide the dividend by",
                              example = "2")
        private int divisor;
    
        public DivideRequest(int dividend, int divisor)
        {
            this.dividend = dividend;
            this.divisor = divisor;
        }
    
        public DivideRequest()
        {
        }
    
        @Override
        @OpenApiRequestHandler(summary = "Divides two numbers")
        public DivideResponse onPost()
        {
            return listenTo(new DivideResponse());
        }
    
        @Override
        public Class<DivideResponse> responseType()
        {
            return DivideResponse.class;
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

Notice that the nested response class uses the outer class to access the request's fields. This makes [getters and setters](https://www.infoworld.com/article/2073723/why-getter-and-setter-methods-are-evil.html) unnecessary. When *onPost()* is called by KivaKit, the response object is created (and any messages it produces are repeated due to the call to *listenTo()*), and the constructor for the *DivideResponse* object performs the divide operation. This makes the *onPost()* handler a one-liner:

		public DivideResponse onPost()
		{
		    return listenTo(new DivideResponse());
		}

Notice how OO design principles have improved encapsulation, eliminated boilerplate and increased readability.

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
                DivideRequest.DivideResponse.class, 
                new DivideRequest(9, 3));
    
            Message.println(AsciiArt.box("response => $", response));
        }
    }

Here, we create a *MicroservletClient* to access the microservice we built above. We tell it to use the service on port 8086 of the local host. Then we POST a *DivideRequest* to divide 9 by 3 using the client, and we read the response. The response shows that the quotient is 3:

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

| Annotation | Purpose |
|------------|---------|
| @OpenApiIncludeMember | Includes the annotated method or field in the specification |
| @OpenApiExcludeMember | Excludes the annotation method or field from the specification |
| @OpenApiIncludeMemberFromSuperType | Includes a member from the superclass or superinterface in the specification |
| @OpenApiIncludeType | Includes the annotated type in the specification schemas |
| @OpenApiRequestHandler | Provides information about a request handling method (*onGet()*, *onPost()* or *onDelete()*) |

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
