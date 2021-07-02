
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.06 - Lazy initialization as a Java design pattern](#lazy)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "lazy"></a>

2021.07.06

### Lazy initialization as a Java design pattern &nbsp; <img src="https://state-of-the-art.org/graphics/coffee/coffee-32.png" srcset="https://state-of-the-art.org/graphics/coffee/coffee-32-2x.png 2x" style="vertical-align:baseline"/>

It would be interesting for Java to have a *lazy* keyword. This imaginary keyword would only evaluate the expression to the right of the equals sign when the reference on the left side is accessed. Code using this keyword might look like this:

    private lazy Map<Key, Value> lazyMap = createMap();
    
    [...]
    
    lazyMap.put(key, value);

Here, just before *lazyMap.put()* is executed, the *createMap()* method would be called and its value assigned to *lazyMap*.

Instead, a common and less succinct idiom in Java is something like this:

    private Map<Key, Value> lazyMap;
    
    [...]
    
    if (lazyMap == null)
    {
        lazyMap = createMap():
    }
    
    lazyMap.put(key, value);

We can't (easily) introduce the *lazy* keyword that we'd like into Java, but we can create a class with similar functionality:

    public class Lazy<Value>
    {
        /**
         * Factory method to create a Lazy object with the given value factory
         */
        public static <V> Lazy<V> of(final Factory<V> factory)
        {
            return new Lazy<>(factory);
        }
    
        /** The value, or null if it doesn't exist */
        private Value value;
    
        /** The factory to create a new value */
        private final Factory<Value> factory;
    
        /**
         * @param factory A factory to create values when needed
         */
        protected Lazy(final Factory<Value> factory)
        {
            this.factory = factory;
        }
        
        /**
         * @return The value
         */
        public synchronized Value get()
        {
            if (value == null)
            {
                value = factory.newInstance();
            }
            return value;
        }
    }

*Lazy* can reduce the four lines of if-null check boilerplate above to these two lines:

    private Map<Key, Value> lazyMap = Lazy.of(this::createMap);
    
    [...]
    
    lazyMap.get().put(key, value);

True, it's not quite as nice as having a *lazy* keyword, but it improves readability by making the flow of code easier to follow. The *Lazy* class is available in kivakit-kernel in the [KivaKit](https://www.kivakit.org) project.
