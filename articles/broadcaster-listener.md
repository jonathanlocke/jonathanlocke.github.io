
#### <img src="https://state-of-the-art.org/kivakit-32.png" srcset="https://state-of-the-art.org/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.13 - How *Broadcaster* / *Listener* loosens coupling and improves component semantics](#broadcaster)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "broadcaster"></a>

2021.07.13

### How *Broadcaster* / *Listener* loosens coupling and improves component semantics  &nbsp; <img src="https://state-of-the-art.org/graphics/sonar/sonar-32.png" srcset="https://state-of-the-art.org/graphics/sonar/sonar-32-2x.png 2x" style="vertical-align:baseline"/>

I'm speculating here, but Dr. Alan Kay's conception of object-oriented programming in the late 1960's might have come about, at least partly, as a result of his undergraduate work in molecular biology. A cell is a pretty good analogy for an object and DNA is a sort of template or class from which cells are created. But to Kay, what would have been the most interesting is cell surface receptors and the way that cells pass various kinds of messages to each other (and themselves) by secreting compounds that bind to these receptors. In fact, he posted this revealing email in 1998:

> [...] Smalltalk is not only NOT its syntax or the class library, it is not even about classes. I'm sorry that I long ago coined the term "objects" for this topic because it gets many people to focus on the lesser idea. The big idea is "messaging" [...] 

Which brings us to the *Broadcaster* / *Listener* design pattern. A language like Java, with statically bound, synchronously invoked methods, is not at all what Dr. Kay is referring to when he talks about messaging. But even if Java isn't a dynamic, late-bound messaging-oriented language, we can still do *some* interesting messaging in Java. The *Broadcaster* / *Listener* design pattern is one way to implement messaging, and it turns out to be very useful and powerful.

A *Broadcaster* transmits *Messages* to an *audience* of one or more *Listeners*:

<img src="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener-300.png" srcset="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener-300-2x.png 2x" style="vertical-align:middle"/>

A *Broadcaster* is a *Transmitter* that keeps a list of *Listener*s (the audience) to which it will transmit messages. Listeners can be added and removed from this audience. When the *Broadcaster.transmit(Transmittable)* method is called with a *Transmittable* message, that message object is transmitted to each member of the audience.

    public interface Transmitter<Transmittable>
    {
        void transmit(Transmittable value);
    }
    
    public interface Broadcaster extends Transmitter<Transmittable>
    {
        /**
         * Adds a listener to this broadcaster that wants to receive 
         * future messages.
         */
        void addListener(Listener listener);
        
        /**
         * Removes the given listener from this broadcaster
         */
        void removeListener(Listener listener);
    }

A *Listener* receives messages transmitted by a broadcaster with its *receive(Transmittable)* method. Different implementations of this interface may do different things with the message.

    public interface Listener<Transmittable>
    {
        /**
         * Registers this listener with the given broadcaster in being 
         * interested in transmitted messages
         */
        default <T extends Broadcaster> T listenTo(T broadcaster)
        {
            broadcaster.addListener(this);
            return broadcaster;
        }
       
        /**
         * Receives the given message
         */ 
        void receive(final Transmittable message);
    }

Here are a few examples of *Listeners* in KivaKit, just to give an idea of the range of tasks that listeners might perform:

| Listener | Purpose |
|---------|--------|
| Logger | Write the message to a Log |
| MutableCount | Counts the number of messages it receives |
| MessageList | Keeps a list of the messages it receives |
| ThrowingListener | Throws an exception containing the message |
| ConsoleWriter | Formats and writes the message to the console |
| StatusPanel | A Swing panel that displays messages it receives |
| NullListener | A listener that discards the messages it receives |
| ValidationReporter | Reports problem messages it receives |

A *Repeater* is both a *Listener* and a *Broadcaster*. When a repeater hears a message via *receive()*, it re-broadcasts it with *transmit()*:

    public interface Repeater extends Listener, Broadcaster
    {
        /**
         * Repeaters broadcast messages they receive
         */
        @Override
        default void receive(final Transmittable message)
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

Here, if the *EmployeeLoader* class can't load employees, it reports a problem via the *problem()* method. The *PayrollProcessor* class creates and listens to an instance of *EmployeeLoader*. Since *PayrollProcessor* is itself a repeater, any messages that the *EmployeeLoader* transmits will re-broadcast. The final line of code, listens to these messages with a *Logger*. This design gives a large amount of flexibility and clean error handling semantics with very little code.

A class can implement the *Repeater* interface into ways. It can extend *BaseRepeater* (as above) or it can implement *RepeaterMixin* if it already has a base class (see the article on this site about Java mixins).

The ability to arbitrarily chain messages, particularly messages that represent some kind of warning or problem, greatly increases component flexibility:

- A class that is a broadcaster or repeater can broadcast problem messages when things go wrong. Rather than throwing an exception to signal a problem, the class can let the caller decide that it wants those semantics by installing a *ThrowingListener*. Rather than returning an error code or an object with error information, the class can let the caller choose to listen for messages with a *MessageList*, allowing it to decide what to do with the messages it captures. The caller might decide that certain problem messages should be logged with a *Logger* or it might want to repeat messages to its own listeners. There's really no limit to the error handling semantics that can be plugged into a component that broadcasts messages. By reporting all problems in a unified way, as broadcast *messages*, our class is simplified and abstracted away from the error handling requirements of the caller.
- Objects that broadcast problem messages don't need to use method return values to provide error information. This can allow methods in the class to make full use of their return value. An object that broadcasts problem messages doesn't need to do anything further in terms of providing error information. It might return a boolean or a value that can be null if it failed, but the description of what went wrong is never a concern.
- It is very typical for long listener chains to terminate in a logger. But that doesn't have to be the case. Messages can be directed to a statistics class that gathers information about how many messages of different types reach the end of the chain. Messages can even be directed to more than one terminal listener. The ability to redirect whole chains of listeners to an arbitrary set of destination(s) is a powerful design.

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

The simplified *Broadcaster* / *Listener* design pattern above is fully implemented in *kivakit-kernel* and is used throughout [KivaKit](https://www.kivakit.org) for messaging between components.
