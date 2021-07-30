
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.08.XX - KivaKit applications and servers](#progress)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "progress"></a>

2021.08.XX

### KivaKit applications and servers &nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/server/server.svg" width="128" style="vertical-align:baseline"/>

The *kivakit-application* module contains building blocks for creating applications and servers in KivaKit. The class *Application* extends *BaseComponent* and *Server* extends *Application*. In turn, *BaseComponent* inherits *Repeater* functionality from *BaseRepeater* and default methods from *Component* (*ComponentMixin*, not shown, also inherits these methods):

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-application.svg" width="600"/>

*Application* provides command-line parsing (see [*kivakit-commandline*](#commandline)) and application lifecycle methods. In addition, it inherits functionality from *BaseComponent* for registering and locating objects and settings (see [*kivakit-configuration*](#configuration)):

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/com.telenav.kivakit.application.component.svg" width="700"/>

#### Line count application example

In the following example, we will create an application that counts the lines of its file argument. With no arguments, the application will give detailed help. With the argument *-show-file-size=true*, it will show the size of the file in bytes.

#### Application initialization

To start our application running we supply code similar to this:

    public class ApplicationExample extends Application
    {
        public static void main(String[] arguments)
        {
            new ApplicationExample().run(arguments);
        }
        
        private ApplicationExample()
        {
            super(ApplicationExampleProject());
        }        
        
        [...]
        
        @Override
        protected void onRun()
        {
            [...]
        }        
    }

The *main()* method creates an instance of the application and the constructor for the application passes an instance of *Project* to the superclass. The application then calls *run()*, which proceeds to initialize the project and application. When the application is fully initialized and ready to run, the  *onRun()* method is called.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/rocket/rocket.svg" width="80"/>

#### Project initialization

To initialize our application's project, we create a subclass of *Project*, which provides any required initialization logic, as well as a set of dependent projects via the *dependencies()* method. When the application runs, its project and all of the sub-projects in its project dependency tree will be initialized before *onRun()* is called:

    public class ApplicationExampleProject extends Project
    {
        private static Lazy<ApplicationExampleProject> project = 
            Lazy.of(ApplicationExampleProject::new);
    
        public static ApplicationExampleProject get()
        {
            return project.get();
        }
    
        protected ApplicationExampleProject()
        {
        }
    
        @Override
        public Set<Project> dependencies()
        {
            return Set.of(ResourceProject.get());
        }
    }

#### Command line parsing and application logic

Once our example application has initialized, *onRun()* provides the application logic:

    private ArgumentParser<File> INPUT =
            fileArgumentParser("Input text file")
                    .required()
                    .build();

    private SwitchParser<Boolean> SHOW_FILE_SIZE =
            booleanSwitchParser("show-file-size", "Show the file size in bytes")
                    .optional()
                    .defaultValue(false)
                    .build();
                    
    @Override
    public String description()
    {
        return "Example application that counts the number of lines" +
               " in the file argument passed to the command line";
    }

    @Override
    protected void onRun()
    {
        var input = argument(INPUT);

        if (input.exists())
        {
            showFile(input);
        }
        else
        {
            problem("File does not exist: $", input.path());
        }
    }
    
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
    
    private void showFile(File input)
    {
        if (get(SHOW_FILE_SIZE))
        {
            information("File size = $", input.sizeInBytes());
        }

        information("Lines = $", Count.count(input.reader().lines()));
    }


When the *Application.run()* method is called, switch and argument parsers for the application are retrieved from *switchParsers()* and *argumentParsers()*, respectively. The logic in *Application.run()* uses these parsers to parse the *String[]* argument to *main()* into a *CommandLine* object.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/command-line/command-line.svg" width="80"/> 

The *onRun()* method in our example application calls *argument()* with the *INPUT* file argument parser to obtain an input *File* object from the *CommandLine*:

    var input = argument(INPUT);

 Next, if the file exists, our example application calls *showFile()* to show the number of lines in the file. In that same method, if the boolean switch *SHOW_FILE_SIZE* is true, it also shows the file size in bytes:
 
    if (get(SHOW_FILE_SIZE))
    {
        information("File size = $", input.sizeInBytes());
    }
    
    information("Lines = $", Count.count(input.reader().lines()));
 
Finally, if something goes wrong interpreting our application's command-line arguments, *KivaKit* will use any broadcast messages as well as information from the argument and switch parsers to provide detailed help about what went wrong and how to correctly use the application:

    ┏━━━━━━━━━━━━━━━━━┫ COMMAND LINE ERROR(S) ┣━━━━━━━━━━━━━━━━━┓
    ┋     ○ Required File argument "Input text file" is missing ┋
    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
     
    
    KivaKit 0.9.9-SNAPSHOT (puffy telephone)
    
    Usage: ApplicationExample 0.9.9-SNAPSHOT <switches> <arguments>
    
    Example application that counts the number of lines in the file argument passed to the command line
    
    Arguments:
    
      1. File (required) - Input text file
    
    Switches:
    
        Optional:
    
      -show-file-size=Boolean (optional, default: false) : Show the file size in bytes
    


The complete code for the example presented here is available in the [*kivakit-examples*](https://github.com/Telenav/kivakit-examples/tree/master/kivakit-examples-application) repository.
