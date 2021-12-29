2021.12.25

### KivaKit Kernel - Message Formatting and Template Expansions &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/nucleus/nucleus.svg" width="48"/>

The module [kivakit-kernel](https://github.com/Telenav/kivakit/tree/master/kivakit-kernel) supports a simple variable substitution syntax. This syntax
can be used when formatting messages, or when substituting variables into templates.

#### Formatting a Message

Basic message formatting is achieved with the *Message.format()* method:

    var formatted = Message.format("Hello my name is $", name);

The symbol *$* is an expansion marker, and the corresponding argument is substituted into the 
formatted string at the marker's location:

    var formatted = format("argument1 = $, argument2 = $", argumentOne, argumentTwo);

Here, *argumentOne*'s string value will be substituted for the first *$* and *argumentTwo*'s string
value will be substituted for the second *$*. Both arguments will be converted to string values using 
*Strings.toString(Object)*. 

<img src="https://www.state-of-the-art.org/graphics/string/string.svg" width="64"/>

In addition to this basic subsitution syntax, arguments can be formatted in a variety of ways using the syntax *${format}*, where *format* is one of the following:

| Format     | Description                                                                                         |
|------------|-----------------------------------------------------------------------------------------------------|
| $$         | evaluates to a literal '$'                                                                          |
| ${class}   | converts a {@link Class} argument to a simple (non-qualified) class name                            |
| ${hex}     | converts a long argument to a hexadecimal value                                                     |
| ${binary}  | converts a long argument to a binary string                                                         |
| ${integer} | converts an integer argument to a non-comma-separated numeric string                                |
| ${long}    | converts a long argument to a non-comma-separated numeric string                                    |
| ${float}   | converts a float argument to a string with one digit after the decimal                              |
| ${double}  | converts a double argument to a string with one digit after the decimal                             |
| ${debug}   | converts an argument to a string using toDebugString() if that interface is supported.              |
| ${object}  | uses the {@link ObjectFormatter} to format the argument by reflecting on it                         |
| ${flag}    | converts boolean arguments to 'enabled' or 'disabled'                                               |
| ${name}    | converts the argument to the name returned by {@link Named#name()} if the argument is {@link Named} |
| ${nowrap}  | signals that the message should not be wrapped                                                      |

For example:

    format("The file named '${name}' was read in $", fileName, start.elapsedSince());

The reason for the ${long} and ${integer} formats in the table above is that *int*, *long* and *Count* objects are formatted by default
with comma separators. This code:

    Count lines;
        [...]
    format("Processed $ lines.", lines);

will produce a string like:

    Processed 1,457,764 lines.

Using *${integer}* like this:

    Count lines;
        [...]
    format("Processed ${integer} lines.", lines);

produces string this instead:

    Processed 1457764 lines.

#### Broadcasting a Message

When [broadcasting a message](2021-07-07-broadcaster.md) from a [component](2021-08-02-components-and-settings.md), 
each message broadcasting method (*information()*, *warning()*, *problem()*, etc.) accepts the same parameters as 
*Message.format()* and they are handled in the same way:

    information("Processed $ lines.", lines);

<img src="https://www.state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener.svg" width="96"/>

#### Templates

The *VariableMap* and *PropertyMap* classes allow for easy template substitutions
using a similar syntax:

    var properties = PropertyMap.load(this, file);
    properties.put("home", "/users/shibo");
    properties.put("job", "/var/jobs/job1");
    var expanded = properties.expand("Home = ${home}, Job = ${job}");

Here, the marker *${home}* is replaced with the value returned by *properties.get("home")*,
and *${job}* is replaced by *properties.get("job")*.

Putting it all together, we can load a *.properties* file and a template, and expand
the template like this:

    Resource template;
    Resource properties;
    var expanded = PropertyMap.load(this, properties).expand(template.string());

The properties file resource *properties* is read with *load()*, broadcasting any warning or
problem messages to *this* object. The property map is then expanded into the template 
read from the resource *template* with *Resource.string()*. Yes, reading and expanding a 
template in KivaKit is a one-liner.

#### Code

The code discussed above is available on GitHub:

 - [kivakit-kernel](https://github.com/Telenav/kivakit/tree/master/kivakit-kernel)  

The KivaKit kernel, is available on *Maven Central* at these coordinates:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-kernel</artifactId>
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
