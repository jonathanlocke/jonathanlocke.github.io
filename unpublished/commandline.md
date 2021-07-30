
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.08.XX - KivaKit command line parsing](#progress)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "progress"></a>

2021.08.XX

### KivaKit command line parsing &nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/command-line/command-line.svg" width="60" style="vertical-align:baseline"/>

The *kivakit-commandline* module provides switch and argument parsing used by *kivakit-application*. When an Application starts up (see [KivaKit applications and servers](#application)), the *Application.run(String[] arguments)* method uses the *kivakit-commandline* module to parse the argument array passed to *main()*:

    public final void run(final String[] arguments)
    {
        onRunning();
        
        [...]

        commandLine = new CommandLineParser(this)
                .addSwitchParsers(switchParsers())
                .addArgumentParsers(argumentParsers())
                .parse(argumentList.asStringArray());

In *run()*, a *CommandLineParser* instance is created and configured with the applications argument and switch parsers, as returned by *switchParsers()* and *argumentParsers()* in the user's application subclass. Next, when the *parse(String[])* method is called, the command line is parsed. The resulting *CommandLine* model is stored in *Application*, and is used later by the user's application subclass to retrieve argument and switch values.

An overview of the classes used in this parsing process can be seen in this abbreviated UML diagram:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-command-line.svg" width="700"/>

The *CommandLineParser* class references a *SwitchParserList* and an *ArgumentParserList*. When its *parse()* method is called, it creates a *CommandLine* object. It parses the switches and arguments from the given string array and populates the *CommandLine*'s *SwitchList* and *ArgumentList* with the resulting switch and argument values. 

Once a *CommandLine* has been successfully parsed, it is available through *commandLine()*. The values of specific arguments and switches can be retrieved through its *get()* and *argument()* methods. The application class provides convenience methods as well so that the call to *commandLine()* can be omitted for the sake of brevity.

#### Example command line parsing

In the example in [KivaKit applications and servers](#application), the argument and switch parsers returned by the example application were declared like this (how these parsers are built with the fluent interface here will be covered below):

    private ArgumentParser<File> INPUT =
            fileArgumentParser("Input text file")
                    .required()
                    .build();

    private SwitchParser<Boolean> SHOW_FILE_SIZE =
            booleanSwitchParser("show-file-size", "Show the file size in bytes")
                    .optional()
                    .defaultValue(false)
                    .build();

In *onRun()*, the input file is retrieved by calling the *argument()* method with the *INPUT* argument parser:

        var input = argument(INPUT);

The SHOW_FILE_SIZE boolean switch is accessed in a similar way with *get()*:

        if (get(SHOW_FILE_SIZE))
        {
            [...]
        }

And finally, the example application has to provide the switch and argument parsers is uses with these two methods:

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

This is *all* that's required to do basic switch parsing in *KivaKit*. 

But there are a few questions left to address. How are arguments and switches validated? How does *KivaKit* automatically provide command line help? And how can we define new *SwitchParser*s and *ArgumentParser*s?

#### Command line validation

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-validation.svg" width="700"/>

#### Command line help

#### Creating new switch and argument parsers



The complete code for the example presented here is available in the [*kivakit-examples*](https://github.com/Telenav/kivakit-examples/tree/master/kivakit-examples-application) repository.

