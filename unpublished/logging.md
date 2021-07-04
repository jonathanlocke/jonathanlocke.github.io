
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.XX - Why the design of KivaKit logging is better than the rest](#converters)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "converters"></a>

2021.07.XX

### Why the design of KivaKit logging is better than the rest &nbsp; <img src="https://state-of-the-art.org/graphics/log/log-32.png" srcset="https://state-of-the-art.org/graphics/log/log-32-2x.png 2x" style="vertical-align:baseline"/>

Logging is a common need for applications and servers to allow events that occurred during execution to be analyzed after the fact. Java has a number of logging frameworks (this is not an exhaustive list):

* Log4J
* Java Logging API
* tinylog
* Logback
* Apache Commons Logging
* SLF4J

KivaKit departs from these frameworks because a KivaKit *Logger* participates in the [*Broadcaster / Listener*](#broadcaster) design pattern because it is a message *Listener*:

    public interface Logger extends Listener
    {
        void log(Message message);
    
        default void onMessage(Message message)
        {
            log(message);
        }
    }

The *Logger.log(Message)* method adds the given message to each *Log* that is attached to the *Logger*. There are a variety of *Logs* available, including:

* *ConsoleLog*
* *EmailLog*
* *FileLog*

Logs can be added from the command line with -DKIVAKIT_LOG. They can be filtered by log level with -DKIVAKIT_LOG_LEVEL, and asynchronous logging can be turned off with -DKIVAKIT_LOG_SYNCHRONOUS=false. For details, see [kivakit system properties](https://github.com/Telenav/kivakit/blob/master/documentation/developing/system-properties.md) and [kivakit-kernel logging](https://github.com/Telenav/kivakit/blob/master/kivakit-kernel/documentation/logging.md).



The *Listener.onMessage(Message)* method simply calls *log()* with any messages that it hears. This is an especially powerful design, because a *Logger* can be plugged in anywhere that a *Listener* is accepted, and listeners are used throughout KivaKit. Messages can be routed through a chain of *Repeater*s and anywhere along the way (or most typically at the end of the chain), they can be logged. The design for the logging frameworks above is more special-purpose where messages are only intended to be logged.

*Logger* instances can be retrieved from *LoggerFactory*:

    public class LoggerFactory
    {
        private static Factory<Logger> factory = LogServiceLogger::new;
    
        public static void factory(Factory<Logger> factory)
        {
            LoggerFactory.factory = factory;
        }
    
        public static Logger newLogger()
        {
            return factory.newInstance();
        }
    }

Typical usage looks like:

    private static final Logger LOGGER = LoggerFactory.newLogger();

However, it is rare to directly use the *Logger* interface. Instead, a component will extend *BaseRepeater* or implement the *RepeaterMixin* interface, and it will broadcast messages. Those messages would then be redirected at the end of the listener chain to a *Logger*. The *Application* class (more on that in a future article) does exactly this:

    public abstract class Application extends BaseRepeater
    {
        private static final Logger LOGGER = LoggerFactory.newLogger();
    
        public final void run(final String[] arguments)
        {
            [...]
        
            LOGGER.listenTo(this);
             
            [...]
        }
        
        [...]
    }

This means that any *Application* subclass that broadcasts a message (directly or through a listener chain) can depend on it being logged (if it isn't filtered out in the listener chain). In practice this looks like:

    public OperationDoomIII extends Application
    {
        void onRun()
        {
            information("Ready to go");
            var attack = listenTo(new Attack(Earth.instance()));
            attack.start();
        }
    }
    
    public Attack extends BaseRepeater
    {
        public void start()
        {
            if (!spaceship.isReady())
            {
                problem("Spaceship isn't ready");
            }
            else
            {
                narrate("Launching in 10 seconds");
                countdown();
            }
        }
        
        [...]
    }

If the spaceship isn't ready when *Attack.start()* is called, a *Problem* message is broadcast. The *OperationDoomIII* class listens to messages from the *Attack*, and the LOGGER in *Application* listens to *OperationDoomIII*, so the message will be logged.

It is worth noting that the methods inherited from *BaseRepeater* allow formatting of messages:

    narrate("Launching in $ seconds", secondsToLaunch());

And the formatting of messages is lazy. If the message is filtered out (for example, by a log level), no formatting occurs. In addition argument evaluation doesn't happen until formatting time, so it's easy to delay evaluation of expensive arguments:

    narrate("Launching in $ seconds", this::computeSecondsToLaunch);

Details on formatting can be found in *MessageFormatter*. 

Registry.of(this)
[KivaKit](https://www.kivakit.org) 
