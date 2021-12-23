2021.12.25

### KivaKit and AWS Lambda &nbsp; <img src="https://www.state-of-the-art.org/graphics/lambda/lambda.svg" width="128"/>

KivaKit 1.2 adds seamless support for [AWS Lambda](https://aws.amazon.com/lambda/). Lambdas for [REST and GRPC](2021-10-01-microservices) 
can be added to a KivaKit Microservice without alteration (which will make this a short article).

#### Creating a Lambda

We have already seen a KivaKit request handler for REST in the [Microservices](2021-10-01-microservices.md) article.
We will simply reuse this code as our Lambda request handler. As a reminder the code from that article looks like this:

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

#### Adding a Lambda Service

In a similar fashion to adding a REST service, a Lambda service is added like this:

    public class DivisionMicroservice extends Microservice
    {
        [...]
    
        @Override
        public MicroserviceLambdaService onNewLambdaService()
        {
            return new DivisionLambdaService(this);
        }
    }

The *onNewLambdaService()* method returns an instance of *DivisionLambdaService*, which extends *MicroserviceLambdaService*:

    public class DivisionLambdaService extends MicroserviceLambdaService
    {
        [...]
    
        @Override
        public void onInitialize()
        {
            mount("division", "1.0", DivisionRequest.class);
        }
    }

When the service is initialized, a call to the *mount()* method in *onInitialize()* is used to associate 
the name of our lambda and its version with the handler *DivisionRequest*. Nothing more is required.

#### Code

The code discussed above is available on GitHub:

 - [kivakit-microservice](https://github.com/Telenav/kivakit-extensions/tree/master/kivakit-microservice)  
 - [kivakit-examples-lambda](https://github.com/Telenav/kivakit-examples/tree/master/kivakit-examples-lambda)  

The KivaKit Microservice API, including support for AWS Lambda, is available on *Maven Central* at these coordinates:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-microservice</artifactId>
        <version>1.2.0</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="lambda"
  theme="github-dark"
  crossorigin="anonymous"
></script>
