
2021.08.19

### KivaKit command line parsing &nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/command-line/command-line.svg" width="50" style="vertical-align:baseline"/>

The *kivakit-commandline* module provides the switch and argument parsing used by *kivakit-application*. Let's take a look at how this works. When an Application starts up (see [KivaKit applications](2021-08-10-applications.md)), the *Application.run(String[] arguments)* method uses the *kivakit-commandline* module to parse the argument array passed to *main()*. Conceptually, this code looks like this:

    public final void run(String[] arguments)
    {
        onRunning();
        
        [...]

        commandLine = new CommandLineParser(this)
                .addSwitchParsers(switchParsers())
                .addArgumentParsers(argumentParsers())
                .parse(arguments);

In *run()*, a *CommandLineParser* instance is created and configured with the applications argument and switch parsers, as returned by *switchParsers()* and *argumentParsers()* in our application subclass. Next, when the *parse(String[])* method is called, the command line is parsed. The resulting *CommandLine* model is stored in *Application*, and is used later by our application to retrieve argument and switch values.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Parsing 

An overview of the classes used in command line parsing can be seen in this abbreviated UML diagram:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-command-line.svg" width="700"/>

The *CommandLineParser* class references a *SwitchParserList* and an *ArgumentParserList*. When its *parse(String[])* method is called, it uses these parsers to parse the switches and arguments from the given string array into a *SwitchList* and an *ArgumentList*. Then it returns a *CommandLine* object populated with these values.

> Note that *all* switches must be of the form -switch-name=[value]. If a string in the argument array is not of this form, it is considered an argument and not a switch.

Once a *CommandLine* has been successfully parsed, it is available through *Application.commandLine()*. The values of specific arguments and switches can be retrieved through its *get()* and *argument()* methods. The *Application* class provides convenience methods so that the call to *commandLine()* can often be omitted for the sake of brevity.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Example

In the example in [*KivaKit applications*](2021-08-10-applications.md), the argument and switch parsers returned by the example application were declared like this:

    import static com.telenav.kivakit.commandline.SwitchParser.booleanSwitchParser;
    import static com.telenav.kivakit.filesystem.File.fileArgumentParser;
    
    [...]
    
    private ArgumentParser<File> INPUT =
            fileArgumentParser("Input text file")
                    .required()
                    .build();

    private SwitchParser<Boolean> SHOW_FILE_SIZE =
            booleanSwitchParser("show-file-size", "Show the file size in bytes")
                    .optional()
                    .defaultValue(false)
                    .build();

The *Application* subclass then provides these parsers to KivaKit like this:

    @Override
    protected List<ArgumentParser<?>> argumentParsers()
    {
        return List.of(INPUT);
    }
    
    @Override
    protected Set<SwitchParser<?>> switchParsers()
    {
        return Set.of(SHOW_FILE_SIZE);
    }

Then in *onRun()*, the input file is retrieved by calling the *argument()* method with the *INPUT* argument parser:

        var input = argument(INPUT);

and the SHOW_FILE_SIZE boolean switch is accessed in a similar way with *get()*:

        if (get(SHOW_FILE_SIZE))
        {
            [...]
        }

This is all that's required to do basic switch parsing in *KivaKit*. 

But there are a few questions to address regarding how all this works. How are arguments and switches validated? How does *KivaKit* automatically provide command line help? And how can we define new *SwitchParser*s and *ArgumentParser*s?

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Command line validation

The [*KivaKit validation*](2021-07-20-validation.md) mini-framework is used to validate switches and arguments. As shown in the diagram below, validators for arguments and switches are implemented in the (private) classes *ArgumentListValidator* and *SwitchListValidator*, respectively. When arguments and switches are parsed by *CommandLineParser* these validators are used to ensure that the resulting parsed values are valid.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-validation.svg" width="700"/>

For the list of switches, *SwitchListValidator* ensures that:

1. No required switches are omitted
2. No switch values are invalid (as determined by the switch parser's validation)
3. No duplicate switches are present (this is not allowed)
4. All switches present are recognized by some switch parser

For the list of arguments, *ArgumentListValidator* ensures that the number of arguments is acceptable. *ArgumentParser.Builder* can specify a quantifier for an argument parser by calling one of these methods:

    public Builder<T> oneOrMore()
    public Builder<T> optional()
    public Builder<T> required()
    public Builder<T> twoOrMore()
    public Builder<T> zeroOrMore()

Argument parsers that accept more than one argument are *only allowed at the end of the list of argument parsers* returned by *Application.argumentParsers()*. For example, this code:

    private static final ArgumentParser<Boolean> RECURSE =
            booleanArgumentParser("True to search recusively")
                    .required()
                    .build();

    private static final ArgumentParser<Folder> ROOT_FOLDER =
            folderArgumentParser("Root folder(s) to search")
                    .oneOrMore()
                    .build();
    
    [...]
    
    @Override
    protected List<ArgumentParser<?>> argumentParsers()
    {
        return List.of(RECURSE, ROOT_FOLDER);
    }

is valid, and will parse command line arguments like this:

    true /usr/bin /var /tmp

Here, each root folder can be retrieved with *Application.argument(int index, ArgumentParser)* passing in the indexes 1, 2 and 3.

However, it would *not* be valid to return these two argument parsers in the reverse order like this:

    @Override
    protected List<ArgumentParser<?>> argumentParsers()
    {
        // NOT ALLOWED
        return List.of(ROOT_FOLDER, RECURSE);
    }

since the ROOT_FOLDER parser must be last in the list.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Command line help

Command line help for applications is provided automatically by KivaKit. For example, forgetting to pass the -deployment switch (more on deployments in a future article) to a server that expects such a switch results in:

    ┏━━━━━━━━━━┫ COMMAND LINE ERROR(S) ┣━━━━━━━━━━┓
    ┋     ○ Required switch -deployment not found ┋
    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
     
    KivaKit 0.9.9-SNAPSHOT (beryllium gorilla)
    
    Usage: DataServer 0.9.0-SNAPSHOT <switches> <arguments>
    
    My cool data server.
    
    Arguments:
    
      <none>
    
    Switches:
    
      Required:
    
      -deployment=Deployment (required) : The deployment configuration to run
    
        ○ localpinot - Pinot on local host
        ○ development - Pinot on pinot-database.mypna.com
        ○ localtest - Test database on local host
      
      Optional:
    
      -port=Integer (optional, default: 8081) : The first port in the range of ports to be allocated
      -quiet=Boolean (optional, default: false) : Minimize output

The description comes from *Application.description()*, which we can override in our application. The argument and switch help is generated from the argument and switch parsers from their name, description, type, quantity, default value, and list of valid values.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Creating new switch and argument parsers

Creating a new switch (or argument) parser is very easy if you have a [*KivaKit type converter*](2021-07-13-converters.md) for the switch. For example, in the application above, we created the *SHOW_FILE_SIZE* switch parser by calling *SwitchParser.booleanSwitchParser()* to create a builder. We then called *optional()* to make the switch optional and gave it a default value of *false* before building the parser with *build()*:

    import static com.telenav.kivakit.commandline.SwitchParser.booleanSwitchParser;

    [...]
    
    private SwitchParser<Boolean> SHOW_FILE_SIZE =
        booleanSwitchParser("show-file-size", "Show file size in bytes")
                .optional()
                .defaultValue(false)
                .build();

The *SwitchParser.booleanSwitchParser* static method creates a *SwitchParser.Builder* like this:

    public static Builder<Boolean> booleanSwitchParser(String name, String description)
    {
        return builder(Boolean.class)
                .name(name)
                .converter(new BooleanConverter(LOGGER))
                .description(description);
    }

As we can see the *Builder.converter(Converter)* method is all that is required to convert the switch from a string on the command line into a *Boolean* value, as in:

    -show-file-size=true
    
In general, if a *StringConverter* already exists for a type, it is trivial to create new switch parser(s) for that type. Because KivaKit has many handy string converters, KivaKit also provides many argument and switch parsers. A few of the types that support switch and/or argument parsers:

* Boolean, Double, Integer, Long
* Minimum, Maximum
* Bytes
* Count
* LocalTime
* Pattern
* Percent
* Version
* Resource, ResourceList
* File, FilePath, FileList
* Folder, FolderList
* Host
* Port

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

The complete code for the example presented here is available in the [*kivakit-examples*](https://github.com/Telenav/kivakit-examples/tree/master/kivakit-examples-application) repository. The switch parsing classes are in:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-commandline</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

but it is not normally necessary to include this directly since the *kivakit-application* module provides easier access to the same functionality:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-application</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="command-line"
  theme="github-dark"
  crossorigin="anonymous"
></script>


