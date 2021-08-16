2021.08.17

### KivaKit logging and the broadcaster / listener design pattern &nbsp; <img src="https://state-of-the-art.org/graphics/log/log-32.png" srcset="https://state-of-the-art.org/graphics/log/log-32-2x.png 2x" style="vertical-align:baseline"/>

Logging allows events that occur during the execution of a program to be analyzed after the fact. A few of the more popular open source logging frameworks for Java include:

* Log4J
* Java Logging API
* tinylog
* Logback
* Apache Commons Logging
* SLF4J

While these frameworks vary in their feature sets, the common idea is that calls to a few specialized logger methods are used to direct log entry information to one or more log implementations. 

KivaKit departs from these frameworks because KivaKit *Logger*s participate in the [*Broadcaster / Listener*](../published/broadcaster.md) design pattern. A *Logger* in KivaKit is a *Listener* which can receive and log any kind of (*Transmittable*) *Message*:

    public interface Logger extends Listener
    {
        void log(Message message);
    
        default void onMessage(Message message)
        {
            log(message);
        }
    }

The *Listener.onMessage(Message)* method here simply calls *log()* with any messages that it hears. This is an especially powerful design, because a *Logger* can be plugged in anywhere that a *Listener* is accepted, and listeners are commonly accepted throughout KivaKit. A message can be routed through a chain of *Repeater*s, and anywhere along the way (although most typically at the end of the chain), the message will be logged if reaches a *Logger*.

*Logger* instances are created by *LoggerFactory*:

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

Usage looks like this:

    private static final Logger LOGGER = LoggerFactory.newLogger();

By default, *LoggerFactory* returns instances of *LogServiceLogger*. This logger parses the KIVAKIT_LOG system property and uses *LogServiceLoader* to dynamically load and configure implementations of the *Log* service provider interface (SPI). Logs are discovered and loaded with Java's *ServiceLoader* class. Once *LogServiceLogger* has loaded and configured its *Log*s, it forwards any messages it receives to each.

A few *Log* implementations are included in KivaKit, and it's very easy to implement new ones:

* *ConsoleLog*
* *EmailLog*
* *FileLog*

Logs can be filtered using the system property KIVAKIT_LOG_LEVEL, and asynchronous logging can be turned off with -DKIVAKIT_LOG_SYNCHRONOUS=false. For more details on logging configuration, see [kivakit-kernel logging](https://github.com/Telenav/kivakit/blob/master/kivakit-kernel/documentation/logging.md). You will need to read this documention enough to understand the details of the KIVAKIT_LOG property at a minimum.

Although *Logger*s are easy enough to use by themselves, in KivaKit it is rare to use one directly. Instead, an object will extend *BaseComponent* or implement the *ComponentMixin* interface, and it will simply broadcast messages without regard to where they might go. Any actual logging only comes in if those messages arrive at a *Logger* at the end of the listener chain (or somewhere along the way). This design completely decouples an object's reporting of its status from logging functionality, while still allowing it to happen when desired.

The *Application* class (more on this in a future article) does exactly this:

    public abstract class Application extends BaseComponent
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

Here, when any *Application* subclass broadcasts a message (directly or through a listener chain) it can depend on it being logged (if it isn't filtered out somewhere along the way in the listener chain). 

In practice usage this looks like:

    public OperationDoomIII extends Application
    {
        void onRun()
        {
            information("Ready to go");
            var attack = listenTo(new Attack(Earth.instance()));
            attack.start();
        }
    }
    
    public Attack extends BaseComponent
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

If the spaceship isn't ready when *Attack.start()* is called, a *Problem* message is broadcast. The *OperationDoomIII* class listens to messages from *Attack*, and the LOGGER in *Application* listens to *OperationDoomIII*, so our *Problem* message, declaring "Spaceship isn't ready", will follow this path:

    Attack.start() -> Attack.Repeater -> OperationDoomIII.Repeater -> Application.LOGGER

It is worth noting that methods like *problem()*, inherited from *BaseComponent*, allow formatting of messages:

    narrate("Launching in $ seconds", secondsToLaunch());

Since the formatting of messages is lazy, if a message is filtered out (perhaps by log level), then no formatting occurs. In addition, argument evaluation doesn't happen until formatting time, so it's easy to delay evaluation of expensive arguments:

    narrate("Launching in $ seconds", this::computeSecondsToLaunch);

Details on formatting can be found in *MessageFormatter*. The logging classes discussed here are part of the kivakit-kernel module of [KivaKit](https://www.kivakit.org).

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
  issue-term="logging"
  theme="github-dark"
  crossorigin="anonymous"
></script>
