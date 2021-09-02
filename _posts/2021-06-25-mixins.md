2021.06.25

### How KivaKit adds mixins to Java  &nbsp; <img src="https://state-of-the-art.org/graphics/math/math-32.png" srcset="https://state-of-the-art.org/graphics/math/math-32-2x.png 2x" style="vertical-align:baseline"/>

[*Traits*](https://tinyurl.com/2n6bbnv3) are a language feature in Scala, Groovy, Kotlin and other languages which allow groups of methods to be added to objects in arbitrary combinations. Unlike interfaces, however, traits can contain method bodies, which allows traits to provide objects with new behaviors. With the addition of [default methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html), Java now provides some of the features of traits, as described by Emil Forslund in [*Traits in Java 8: Semantic, DRY-compliant, Interface-first Code*](https://dzone.com/articles/definition-of-the-trait-pattern-in-java).

When traits include state, they are referred to as *stateful traits* or *mixins*. It is also possible to implement mixins in Java:

    public interface Mixin
    {
        /**
         * Retrieves state associated with the given Mixin sub-interface. 
         * This is done in MixinState using a composite key formed
         * from this object's *this* reference and the type of the Mixin 
         * sub-interface.
         *
         * @param type The sub-interface's type (like AttributedMixin.class)
         * @param factory A factory that creates the initial state to
         * associate with this mixin
         * @return The state for this mixin
         */
        default <T> T state(Class<? extends Mixin> type, Factory<T> factory)
        {
            // Retrieve the state for this mixin that is attached to this object
            return MixinState.get(this, type, factory);
        }
    }

Here, the *state()* method allows any interface extending *Mixin* to retrieve associated state for itself. If the state is not yet attached to the mixin, it is created with the given *Factory*. The *MixinState* class that is used in the body of the *state()* method stores and retrieves state using a key that combines the object's *this* reference and the *Class* of the mixin sub-interface:

    class MixinState
    {
        private static final Map<MixinKey, Object> values = new ConcurrentHashMap<>();
    
        /**
         * @return Gets a value for the given mixin of the given object, 
         * creating a new value with the given factory if it doesn't 
         * already exist.
         */
        public static synchronized <T> T get(final Object object,
                                             final Class<? extends Mixin> mixinType,
                                             final Factory<T> factory)
        {
            // Create a composite key (to allow multiple traits on an object),
            final var key = new MixinKey(object, mixinType);
    
            // get any current value for the mixin,
            var value = (T) values.get(key);
    
            // and if none exists,
            if (value == null)
            {
                // create a new value,
                value = factory.newInstance();
    
                // and store that.
                values.put(key, value);
            }
    
            return value;
        }

This allows each object to have state for many mixins associated with it. Note that an object using mixin(s) does not need to implement the *hashCode()* / *equals()* contract. Also note that mixin state is stored by the package private class *MixinState* using a *ConcurrentHashMap* to avoid concurrency problems. Access to this map may become contentious if there are a large number of objects accessing mixin state at the same time.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Example

To define a new *Mixin*, we can now simply extend the *Mixin* interface and use the *state()* method in our interface's default methods to access our state, as needed:

    public interface AttributedMixin<Key, Value> extends Mixin
    {
        default Value attribute(Key key)
        {
            return map().get(key);
        }
    
        default Value attribute(Key key, Value value)
        {
            return map().put(key, value);
        }
    
        private HashMap<Key, Value> map()
        {
            return state(AttributedMixin.class, () -> new HashMap<>());
        }
    }

Our *AttributedMixin* can now be added to any object, and it will provide a keyed attribute attached to that object. For example, it can be used like this to add a name to any arbitrary class:

    public class AttributedMixinTest
    {
        static class A implements AttributedMixin<String, String> { }
    
        static class B implements AttributedMixin<String, String> { }
    
        @Test
        public void test()
        {
            final var a = new A();
            final var b = new B();
    
            a.attribute("name", "This is object A");
            b.attribute("name", "This is object B");
    
            ensureEqual("This is object A", a.attribute("name"));
            ensureEqual("This is object B", b.attribute("name"));
        }
    }

This is the unit test for *AttributedMixin*.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

Mixins are available in the *kivakit-kernel* project, which is a module in [KivaKit](https://www.kivakit.org):

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
  issue-term="mixins"
  theme="github-dark"
  crossorigin="anonymous"
></script>

