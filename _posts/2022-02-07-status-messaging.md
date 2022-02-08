
2022.02.07

### Why Messaging is a Better Way to Report Status &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/gears/gears.svg" width="48"/>

In a typical Java program, the status of a method call is typically reported in one of three ways:

1. The method returns a value containing status information
2. The method throws an exception containing status information
3. The method logs or otherwise stores status information

One result of this is that every API has different error reporting semantics. 
We have all seen code similar to these examples:

#### Status Reporting via Return Values

```
var sender = new EmailSender();

var result = sender.send(email);
if (result != null)
{
    logger.error(result);
}

if (!sender.send(email))
{
    logger.error("Could not send");
}

var error = sender.send(email);
if (error < 0)
{
    logger.error("Couldn't send email: Error #" + error);
}
```

#### Status Reporting via Exceptions

```
var sender = new EmailSender();

try
{
    sender.send(email);
}
catch (SendException e)
{
    logger.error("Unable to send", e);
}

try
{
    sender.send(email);
}
catch (SendException e)
{
    throw new EmailException("Couldn't send email", e);
}
```

<img src="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

An alternative to these common idioms is to report status "out-of-band" by broadcasting messages to interested listeners. This has a few immediate advantages:

- It standardizes error reporting, allowing API users to treat all components in the same way, reducing the learning curve for new APIs
- It allows return values and exceptions to be used only for control flow
- It simplifies and clarifies the responsibilities of components by decoupling status reporting from status handling
- It allows multiple listeners and chains of listeners to handle the same status information in different ways

<img src="https://state-of-the-art.org/graphics/link/link-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

This status reporting model is implemented by the Java Open Source microservices 
framework, [KivaKit](https://www.kivakit.org). In KivaKit, our *EmailSender* class would implement *Repeater* 
(which is a kind of *Broadcaster*) by extending *BaseRepeater* or (more likely) 
*BaseComponent*.

```
class EmailSender extends BaseRepeater
{
    Connector connector = listenTo(new Connector());
    
    boolean send(Email email)
    {
        if (!connector.connect())
        {
            problem("Could not send");
            return false;
        }

            [...]
            
        return true;
    }
}

class Connector extends BaseRepeater
{
    boolean connect()
    {
        if ( [...] )
        {
            problem("Cannot connect to server");
        }
    }
}

class Client extends BaseRepeater
{
    EmailSender sender = listenTo(new EmailSender());

    void sendReport()
    {
        sender.send(composeReport());
    }
}
```
Here, we can see that the *Connector* object created during construction is 
passed to *listenTo()*. The *listenTo()* method, inherited from *BaseRepeater*, 
causes *EmailSender* to listen for any messages broadcast by *Connector*. Since 
*EmailSender* is a *Repeater*, it will repeat any messages it receives from 
*Connector* to its own listeners in *Client*, forming a listener chain:

```
Connector ==> EmailSender ==> Client ==> [...]
```

> Note: In KivaKit, broadcasting messages does not involve message queueing or 
concurrency. Messages are delivered to listeners synchronously via the method 
Listener.onMessage(). 
 
What's important here is that *Connector* and *EmailSender* 
only report problems regarding their own status. If *Connector* is unable to 
*connect()*, it simply calls *problem()* explaining what it couldn't do. The 
*problem()* method then transmits a *Problem* message to *EmailSender*, and 
*EmailSender* in turn transmits it to *Client*. In this way, *EmailSender* 
indirectly provides "out-of-band" information about *Connector*'s status to 
*Client*. Of course, *EmailSender* also broadcasts its own *Problem* message 
if the *Connector.connect()* method fails, without needing to worry about 
reporting why the connection failed.

The flow of control for KivaKit messaging is shown in this UML sequence diagram:

<img src="https://state-of-the-art.org/uml/out-of-band.png" style="display: block; margin-left: auto; margin-right: auto;"/>

<div style="text-align: center; font-size: 12px">KivaKit's "Out-of-Band" Messaging</div>

<br/>

*Client* calls *EmailSender.send()*, which calls *Connector.connect()*. During the 
execution of each of these methods, status messages may be transmitted down 
the listener chain (as shown by the orange lines) when a method like *problem()*
is called.

<img src="https://state-of-the-art.org/graphics/ruler/ruler-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

The rules of listener chains are:

- *Components* are *Repeaters* which *listenTo()* any components they create to form a listener chain
- Each component in the chain minds its own business, broadcasting any relevant status messages during method calls
- *Listeners* can be added at any point along the chain to capture and process status information

Use of EmailSender in Client now looks like this:

```
var sender = listenTo(new EmailSender());
if (!sender.send(email))
{
    problem("Unable to send report: $", email);
    return false;
}
return true;
```

<img src="https://state-of-the-art.org/graphics/compress/compress-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

Because the semantics of KivaKit status reporting are so regular, it's possible to condense this idiom even further:

```
return isTrueOr(sender.send(), "Unable to send report $", email);
```

The *isTrueOr()* method here is inherited from the *LanguageTrait* interface 
via the *Component* interface. If the boolean argument to *isTrueOr()* is not 
true, it broadcasts the message specified by the remaining arguments as a 
*Problem*. It then returns the boolean value. This eliminates a lot of 
boilerplate if/else nesting when checking multiple conditions. Code like 
this (23 lines):

```
if (isTimeToSend())
{
    if (email != null)
    {
        if (sender.send())
        {
            information("Email sent.");
            return true;
        }
        else
        {
            problem("Unable to send report");
        }
    }
    else
    {
        problem("No email to send");
    }
}
else
{
    problem("Not time to send yet");
}
```

can be reduced to just this (8 lines):

```
if (isTrueOr(isTimeToSend(), "Not time to send yet") &&
    isNonNullOr(email, "No email to send") &&
    isTrueOr(sender.send(email), "Unable to send report"))
{
    information("Email sent.");
    return true;
}
return false;
```

<img src="https://state-of-the-art.org/graphics/mirror/mirror-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

#### A Few KivaKit Listeners and Repeaters

KivaKit has hundreds of *Listeners* and *Repeaters*. This includes all KivaKit 
*Components*. To give an idea of the range of *Listener* implementations:

| Listener              | Purpose                                         |
|-----------------------|-------------------------------------------------|
| *Logger*              | Log status messages                             |
| *MutableCount*        | Count status messages                           |
| *MessageAlarm*        | Trigger an alarm if there are too many problems |
| *StatusPanel*         | Show status messages in a Swing panel           |
| *MessageList*         | Capture messages in a list                      |
| *ValidationIssues*    | Capture problems during validation              |
| *Converter*           | Broadcast conversion errors                     |
| *ThrowingListener*    | Throws an exception when a problem is received  |
| *ConsoleWriter*       | Writes status messages directly to the console  |
| *MicroservletRequest* | Captures microservlet request handling errors   |
| *Application*         | Terminal listener that logs and counts messages |

<img src="https://state-of-the-art.org/graphics/footprints/footprints-96.png" style="display: block; margin-left: auto; margin-right: auto;"/>

### Conclusion

In this short article we took a look at the advantages of broadcaster/listener 
messaging in status reporting over more typical idioms. We then took a look at 
how KivaKit implements this system. More information on KivaKit messaging is 
available [on my blog](https://state-of-the-art.org/2021/07/07/broadcaster.html) 
as well as on [GitHub](https://github.com/Telenav/kivakit/blob/master/kivakit-kernel/documentation/messaging.md).

<br/>
<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="status-messaging"
  theme="github-dark"
  crossorigin="anonymous"
></script>
