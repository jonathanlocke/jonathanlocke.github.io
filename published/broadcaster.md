2021.07.07

### How *Broadcaster / Listener* improves component semantics &nbsp; <img src="https://state-of-the-art.org/graphics/sonar/sonar.svg" width="22" style="vertical-align:baseline"/>

Dr. Alan Kay's conception of object-oriented programming in the late 1960's came about, in part, as a result of his undergraduate work in molecular biology. A cell is a pretty good analogy for an object and DNA is a sort of template or class from which cells are created. But to Kay, what would have been most interesting were cell surface receptors, and the way that cells pass various kinds of messages to each other (and themselves) by secreting compounds that bind to these receptors. He posted this revealing email in 1998:

> [...] Smalltalk is not only NOT its syntax or the class library, it is not even about classes. I'm sorry that I long ago coined the term "objects" for this topic because it gets many people to focus on the lesser idea. The big idea is "messaging" [...] 

Which brings us to the *Broadcaster / Listener* design pattern. A language like Java, with statically bound, synchronously invoked methods, is not at all what Dr. Kay means when he talks about messaging. But even if Java isn't a dynamic, late-bound, messaging-oriented language, we can still do some interesting messaging in Java. The *Broadcaster / Listener* design pattern is one way to do it, and it turns out to be very useful and powerful.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Broadcasters

A *Broadcaster* is a *Transmitter* that transmits [*Messages*](../uml/diagram-message-type.svg) to an audience of one or more *Listeners*. Here are a few of KivaKits most commonly used messages:

| Message | Purpose |
|---|---|
| CriticalAlert | An operation failed and needs *immediate attention* from a human operator | 
| Alert | An operation failed and needs to be looked at by an operator soon |
| FatalProblem | An unrecoverable problem has caused an operation to fail and needs to be addressed |
| Problem | Something has gone wrong and needs to be addressed, but it's not fatal to the current operation |
| Glitch | A minor problem has occurred. Unlike a Warning, a Glitch indicates validation failure or minor data loss has occurred. Unlike a Problem, a Glitch indicates that the operation will definitely recover and continue. |
| Warning | A minor issue occurred but does not necessarily require attention |
| Narration | A step in some operation has started or completed |
| Information | Commonly useful information that doesn't represent any problem |
| Trace | Diagnostic information for use when debugging |

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener.svg" width="120" style="vertical-align:middle"/>

A [*Broadcaster*](../uml/diagram-message-broadcaster.svg) keeps a list of *Listener*s (the audience) to which it will transmit messages. Listeners can be added and removed from the list. When the *transmit(Transmittable)* method is called with a message, that message is given to each member of the audience:

    public interface Transmitter<T extends Transmittable>
    {
        void transmit(T value);
    }
    
    public interface Broadcaster extends Transmitter<Transmittable>
    {
        void addListener(Listener listener);
        void removeListener(Listener listener);
    }

> Note: By default, a broadcaster with no listeners will log any messages it hears, along with a warning that the broadcaster has no listeners. The system property *KIVAKIT_IGNORE_MISSING_LISTENERS* can be used to suppress these warnings for applications where this is acceptable.

<br/>
<img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Listeners

A [*Listener*](../uml/diagram-message-listener.svg) receives messages sent to it with its *receive(Transmittable)* method. Different implementations of this interface may do different things with the message:

    public interface Listener<Transmittable>
    {
        default <T extends Broadcaster> T listenTo(T broadcaster)
        {
            broadcaster.addListener(this);
            return broadcaster;
        }
       
        void receive(Transmittable message);
    }

Here are a few examples of the many [*Listener*s](../uml/diagram-message-listener-type.svg) in KivaKit, just to give an idea of the range of tasks that listeners can perform:

| Listener | Purpose |
|---------|--------|
| Logger | Write the message to a Log |
| MutableCount | Counts the number of messages it receives |
| MessageList | Keeps a list of the messages it receives |
| ThrowingListener | Throws an exception containing the message |
| ConsoleWriter | Formats and writes the message to the console |
| StatusPanel | A Swing panel that displays messages it receives |
| NullListener | A listener that discards the messages it receives |
| ValidationReporter | Reports validation problems it receives |

<br/>
<img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Repeaters

A [*Repeater*](../uml/diagram-message-repeater.svg) is both a *Listener* and a *Broadcaster*. When a repeater hears a message via *receive()*, it re-broadcasts it with *transmit()*:

    public interface Repeater extends Listener, Broadcaster
    {
        @Override
        default void receive(Transmittable message)
        {
            transmit(message);
        }
    }

This process of listening and re-broadcasting forms a *listener chain*:

    broadcaster -> repeater -> listener

A repeater can, of course, listen to other repeaters, forming longer listener chains:

    broadcaster -> repeater -> repeater -> repeater -> listener

For example:

    class EmployeeLoader extends BaseRepeater
    {
        [...]
         
        problem("Unable to load $", employee);
    
        [...]
    }
    
    class PayrollProcessor extends BaseRepeater
    {
         [...]
    
         var loader = listenTo(new EmployeeLoader());
         
         [...]
    }
    
    var processor = LOGGER.listenTo(new PayrollProcessor());

Here, if the *EmployeeLoader* class can't load employees, it reports a problem via the *problem()* method. The *problem()* method in *BaseRepeater* broadcasts a *Problem* message to all listeners of *EmployeeLoader*. The *PayrollProcessor* class creates and listens to an instance of *EmployeeLoader*. Since *PayrollProcessor* is itself a repeater, any messages that the *EmployeeLoader* broadcasts to *PayrollProcessor* will be re-broadcast by *PayrollProcessor* to each of its listeners. 

The final line of code listens to messages from *PayrollProcessor* with a *Logger*. So if something goes wrong loading employees, the problem message goes from *EmployeeLoader* through *PayrollProcessor* to *Logger*, where it gets logged:

    EmployeeLoader -> PayrollProcessor -> Logger

This design yields a lots of flexibility as well as clean, simple error handling with very little code.

A class can implement the *Repeater* interface in two ways. It can extend *BaseRepeater* (as above) or it can implement *RepeaterMixin* if it already has a base class. For more details on how mixins work in KivaKit, see [*how KivaKit adds mixins to Java.*](../published/mixins.md)

The ability to arbitrarily chain messages, particularly messages that represent some kind of warning or problem, greatly increases component flexibility:

- A class that is a *Broadcaster* or *Repeater* can broadcast problem messages when things go wrong. Rather than throwing an exception to signal a problem, the class can let the caller decide that it wants those semantics by installing a *ThrowingListener*. Rather than returning an error code or an object with error information, the class can let the caller choose to listen for messages with a *MessageList*, allowing it to decide what to do with the messages it captures. The caller might decide that certain problem messages should be logged with a *Logger*, or it might just want to repeat messages to its own listeners. There's really no limit to the error handling semantics that can be plugged into a component that broadcasts its status with messages. **By reporting all problems in a unified way as *messages*, our class is simplified and abstracted away from the error handling requirements of the caller**.

- **Objects that broadcast problem messages don't need to use method return values to provide error information. This allows methods to make full use of their return values.** An object that broadcasts problem messages doesn't need to do anything further in terms of providing error information (it can even create new messages that include additional information). It might return a *boolean* or a value that can be *null* if the method failed, but the description of what went wrong (or where or when) is never a concern.

- It is very typical for long listener chains to simply terminate with a *Logger*. But that doesn't have to be the case. Messages might be directed, for example, to a statistics class that gathers information about how many messages of different types reach the end of the chain. Messages might even be directed to more than one terminal listener. **The ability to redirect whole chains of listeners to an arbitrary set of destination(s) is a powerful design**.

KivaKit has hundreds of classes that are *Repeaters*. This allows most non-trivial components to be chained together with ease. A short, randomly selected list of classes in KivaKit that are repeaters:

| Repeater | Purpose |
|---------|---------|
| Converter | Convert one type to another |
| Resource | Provides access to streamed resources |
| EmailSender | Sends emails |
| Application | Base application class that logs messages by default |
| ConnectionListener | Listens for server connections |
| JarLauncher | Downloads and launches a JAR file |

Each of these classes listens to errors from components it uses and broadcasts problems to its own listeners.

<br/>
<img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code 

The simplified *Broadcaster / Listener* design pattern discussed above is available in its entirety in *kivakit-kernel* and is used throughout [KivaKit](https://www.kivakit.org) for messaging between components:

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
  issue-term="broadcaster"
  theme="github-dark"
  crossorigin="anonymous"
></script>
