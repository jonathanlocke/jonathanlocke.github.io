2021.11.12

### KivaKit XML Streaming &nbsp; <img src="https://www.state-of-the-art.org/graphics/tag/tag.svg" width="40"/>

Since Java 1.6 in 2006, Java has had a built-in XML streaming API in the package *javax.xml.stream*. This API is known as StAX (Streaming API for XML), and it is a very efficient "pull parser", allowing clients to iterate through the sequence of elements in an XML document. Other approaches to working with XML are event-handling "push parsers", and full-blown, in-memory DOMs (Document Object Models). Although StAX is convenient and very fast, it can be significantly harder to work with than a DOM, because the hierarchy of the document that's being streamed is lost. Our code sees just one element at a time.

#### KivaKit's New XML Mini-framework

KivaKit 1.1 quietly added a small, but useful mini-framework to the *kivakit-extensions* repository called *kivakit-data-formats-xml*. The project contains just two simple classes: *StaxReader* and *StaxPath*. The *StaxReader* class adds a layer of convenience to the Java StAX API by making it easy to:

* Open and close XML streams
* Get information about the reader's stream position
* Advance through XML elements
* Determine the reader's hierarchical positioning in the stream

#### Moving Through an XML Stream

The static *StaxReader.open(Resource)* method is used to start reading an XML stream. The method either returns a valid *StaxReader* that's ready to go, or it throws an exception. Since *StaxReader* implements *Closeable*, it can be used within a *try-with-resources* statement:

    try (var reader = StaxReader.read(file))
    {
        [...]
    }

Within our *try-with-resources* block, we can advance through the stream with these methods:

* *hasNext()*
* *next()*
* *at()*
* *nextAttribute()*
* *nextCharacters()*
* *nextOpenTag()*
* *nextCloseTag()*
* *nextMatching(Matcher)*

When we reach the end of the stream, *hasNext()* will return false. So, processing an XML file looks like this:

    try (var reader = StaxReader.read(file))
    {
        for (; reader.hasNext(); reader.next())
        {
            var element = reader.at();
            
            [...]
            
        }        
    }

As we stream our way through an XML document, a few simple methods can help us to identify what kind of tag the reader is currently *at*:

* *isAtEnd()*
* *isAtCharacters()*
* *isAtOpenTag()*
* *isAtCloseTag()*
* *isAtOpenCloseTag()*

#### Streaming Through an XML Hierarchy

Although the underlying StAX API can only move through a document in sequential order, *StaxReader* adds functionality that allows us to determine where we are in the document hierarchy as we move along. Using the hierarchical path to our current position the stream, we can search for specific elements in the nested document structure, and we can process data when we reach those elements.

Okay. Let's make this concrete. Here is a simple document:

    <a>   <---- The path here is a
        <b>   <---- The path here is a/b
            <c>   <---- The path here is a/b/c
            </c>
        </b>
    </a>

The *StaxPath* class represents hierarchical paths in our XML document. As seen above, the path at &lt;a&gt; is *a*. The path at &lt;b&gt; in our document is *a/b* and the path at &lt;c&gt; is *a/b/c*. 

As we read elements from the XML stream, *StaxReader* tracks the current path using a stack of elements. When the reader encounters an open tag, it pushes the name of the tag onto the end of the current path. When it encounters a close tag, it pops the last element off the end of the path. So, the sequence of steps as *StaxReader* streams through our document is:

    Step  Element     Action    StaxPath
    1.    <a>         push a    a
    2.      <b>       push b    a/b
    3.        <c>     push c    a/b/c
    4.        </c>    pop       a/b
    5.      </b>      pop       a
    6.    </a>        pop       

The current *StaxPath* for a *StaxReader* can be obtained by calling *StaxReader.path()*.

#### Finding Elements in the Document Hierarchy

The following methods test the current path of our *StaxReader* reader against a given path. The reader is considered *at* a given path if the the reader's path is equal to the given path. For example, if the reader is at the path a/b/c and the given path is a/b/c, the reader is *at* the given path. The reader is *inside* the given path if the given path is a prefix of the reader's current path. For example,
if the reader is at a/b/c/d and the path is a/b/c, then the reader is *inside* the given path. Finally, the reader is *outside* the given path in the reverse situation, where the reader is at a/b and the path is a/b/c.

* *isAt(StaxPath)* - Returns true if the reader is at the given path
* *findNext(StaxPath)* - Advances until the given path or the end of the document is reached
* *isInside(StaxPath)* - Returns true if the reader is *inside* the given path
* *isAtOrInside(StaxPath)* - Returns true if the reader is *at* or *inside* the given path
* *isOutside(StaxPath)* - Returns true if the reader is *outside* the given path.

#### Putting it All Together: Fiasco

And now, the reason for doing all of this. I am working on a build tool for Java projects called [Fiasco](https://github.com/Telenav/fiasco/tree/develop) (named for Stanislaw Lem's 1986 science fiction novel, [Fiasko](https://en.wikipedia.org/wiki/Fiasco_(novel))).

Fiasco is in the design stages still. When it is completed, it will be  a pure-Java build tool with these design goals, and non-goals:

##### Goals

 - Intuitive API
 - Quick learning curve
 - Easy to understand and debug builds
 - Modular design
 - Easy to create new build tools
 - Reasonably fast

##### Non-Goals

 - Incremental compilation
 - Build security

##### How Fiasco Reads POM Files with StaxReader

To build a Java project, Fiasco needs to access artifacts in Maven repositories, like *Maven Central*. To be able to do this, it is necessary to parse Maven pom.xml files, and since Fiasco will be parsing a *lot* of these files, it is desirable to do it fairly efficiently. The [*PomReader*](https://github.com/Telenav/fiasco/blob/feature/fiasco-resolver/src/main/java/com/telenav/fiasco/internal/building/dependencies/pom/PomReader.java) class demonstrates how *StaxReader* can be used to parse a complex, hierarchical XML file like a Maven POM file.
The relevant details are found in the method *read(MavenRepository, Resource)*, which opens a *pom.xml* resource, parses it and returns a *Pom* model:

    public class PomReader extends BaseComponent
    {
        StaxPath PROPERTIES_PATH = StaxPath.parseXmlPath("project/properties");
        StaxPath DEPENDENCY_PATH = StaxPath.parseXmlPath("project/dependencies/dependency");
    
        [...]
        
        Pom read(MavenRepository repository, Resource resource)
        {
            [...]
            
            try (var reader = StaxReader.open(resource))
            {
                var pom = new Pom(resource);
    
                for (reader.next(); reader.hasNext(); reader.next())
                {
                    [...]
                    
                    if (reader.isAt(PROPERTIES_PATH))
                    {
                        pom.properties = readProperties(reader);
                    }
    
                    if (reader.isAt(DEPENDENCY_PATH))
                    {
                        pom.dependencies.add(readDependency(reader));
                    }
                    
                    [...]
                 }
             }
             
         [...]        

     }
        
The code here that opens our *pom.xml* resource and moves through the elements of the XML document is essentially the same as we saw earlier. Within the *for* loop, is where we deal with the POM hierarchy. The *StaxReader.isAt(StaxPath)* method is used to determine when the reader lands on the open tag for the given path. When PROPERTIES_PATH (project/properties) is reached, we call a method that reads the properties nested within the &lt;properties&gt; open tag (see *readProperties(StaxReader)* for the gory details). For each DEPENDENCY_PATH (project/dependencies/dependency) we reach (there may be many of them), we read the &lt;dependency&gt; with logic similar to this:

    Dependency readDependency(StaxReader reader)
    {
        MavenArtifactGroup artifactGroup = null;
        String artifactIdentifier = null;
        String version = null;
        var scope = DEPENDENCY_PATH;

        // Skip past the <dependency> open tag we landed on,
        reader.next();

        // and while we're not outside the <dependency> tag scope,
        for (; !reader.isOutside(scope); reader.next())
        {
            // populate any group id,
            if (reader.isAt(scope.withChild("groupId")))
            {
                artifactGroup = MavenArtifactGroup.parse(this, reader.enclosedText());
            }

            // any artifact id,
            if (reader.isAt(scope.withChild("artifactId")))
            {
                artifactIdentifier = reader.enclosedText();
            }

            // and any version.
            if (reader.isAt(scope.withChild("version")))
            {
                version = reader.enclosedText();
            }
        }

So, there we have it. It is true that our code here is not as succinct as it would be with a DOM model. However, we can parse POM files well enough with *StaxReader*, and we save the time and memory required by a full, in-memory DOM model.
     
#### Code

The code discussed above is available on GitHub:

[kivakit-data-formats-xml](https://github.com/Telenav/kivakit-extensions/tree/0b0389677d3a82cb0378cf9cb5bd8f67852daeda/kivakit-data/formats/xml)  
[Fiasco (GitHub)](https://github.com/Telenav/fiasco/tree/develop)  
[PomReader.java](https://github.com/Telenav/kivakit-extensions/blob/0b0389677d3a82cb0378cf9cb5bd8f67852daeda/kivakit-data/formats/xml/src/main/java/com/telenav/kivakit/data/formats/xml/stax/StaxReader.java)

The KivaKit XML API is available on *Maven Central* at these coordinates:

    <dependency>
        <groupId>com.telenav.kivakit</groupId>
        <artifactId>kivakit-data-formats-xml</artifactId>
        <version>${kivakit.version}</version>
    </dependency>

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="xml-streaming"
  theme="github-dark"
  crossorigin="anonymous"
></script>
