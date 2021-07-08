
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.XX.XX - KivaKit resources](#resources)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "state-machine"></a>

2021.XX.XX

### KivaKit resources &nbsp; <img src="https://state-of-the-art.org/graphics/water/water-32.png" srcset="https://state-of-the-art.org/graphics/water/water-32-2x.png 2x" style="vertical-align:baseline"/>

A resource is a stream of data that can be opened and then read from or written to. Examples of resources include:

* Files
* Sockets
* Zip or JAR file entries
* S3 objects
* HDFS files
* HTTP responses
* Input streams
* Output streams

KivaKit provides an abstraction that allows easy and consistent access to these resources and others, and it is easy to create new resources. Some short examples that use the resource mini-framework API:

*Read the lines of a .csv file from a package, reporting progress:*

    var resource = PackageResource.of(getClass(), "target-planets.csv");
    try (var line : listenTo(new CsvReader(resource, schema, ',', reporter)).lines())
    {
        [...]
    }

*Write a string to a file on S3:*

    var file = File.parse("s3://mybucket/myobject.txt");    
    try (var out = listenTo(file).writer().printWriter())
    {
        out.println("Start Operation Impending Doom III in 10 seconds");
    }

*Safely extract an entry (ensuring no partial result) from a .zip file:*

    var file = File.parse("/users/jonathan/input.zip");
    var folder = Folder.parse("/users/jonathan");
    try (ZipArchive zip = ZipArchive.open(file, reporter, READ))
    {
        listenTo(zip.entry("data.txt")).safeCopyTo(folder, OVERWRITE);
    }
    
In each case, the code is assumed to be in a *Repeater* class. So, the *listenTo()* call will broadcast messages to the listener of that class. Each example will throw an exception if the resource cannot be opened or read from.

The design of KivaKit resources is [fairly complex](https://www.kivakit.org/0.9.8-beta/lexakai/kivakit/kivakit-resource/documentation/diagrams/diagram-resource.svg), so we will focus on the most important, high level aspects in this article.

A simplified UML diagram:

![](../uml/simplified-resource-uml.png)

The *Resource* class in this diagram is central. This class:

* Has a *ResourcePath* (from *ResourcePathed*)
* Has a size in bytes (from *ByteSized*)
* Has a time of last modification (from *ModificationTimestamped*)
* Is a *ReadableResource*

Since all resources are *ReadableResource*s, they can be opened with *Readable.openForReading()* or read from with the convenience methods  *ResourceReader*.

In addition, some resources are *WritableResource*s. Those can be opened with *Writable.openForWriting()* and written to with the convenience class *ResourceWriter*

The *Resource* class itself can determine if the resource *exists()* and if it *isRemote()*. Remote resources can be *materialized* to the local filesystem before reading them (using methods not in the UML diagram). *Resource*s can perform a safe copy to a destination *File* or *Folder* with the two *safeCopyTo()* methods. Safe copying involves 3 steps:

1. Write to a temporary file
2. Delete the destination file
3. Rename the temporary file to the destination filename

Finally, *BaseWritableResource* extends *BaseReadableResource* to add the ability to *delete* a resource, and to save an *InputStream* to the resource, reporting progress as it does this. This is the class hierarchy for *ReadableResource*s and *WritableResource*s:

<img src="../uml/resource-classes.png" width=500/>

Notice that all resources are *Repeater*s (see [Broadcaster/Listener design pattern](#broadcaster)), which makes error handling easy and consistent. 

The implementation of a simple *ReadableResource* requires only an *onOpenForReading* method and a *sizeInBytes()* method. The *StringResource* class is a good example, and looks like this:

    public class StringResource extends BaseReadableResource
    {
        private final String value;
    
        public StringResource(final ResourcePath path, final String value)
        {
            super(path);
            this.value = value;
        }
    
        @Override
        public InputStream onOpenForReading()
        {
            return new StringInput(value);
        }
    
            @Override
        public Bytes sizeInBytes()
        {
            return Bytes.bytes(value.length());
        }
    }



Some things we didn't talk about:

* All resources are repeaters 
* All resources transparently implement compression and decompression


[KivaKit](https://www.kivakit.org)
