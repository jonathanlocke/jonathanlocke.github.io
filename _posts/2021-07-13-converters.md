2021.07.13

### KivaKit type converters &nbsp; <img src="https://state-of-the-art.org/graphics/wand/wand-32.png" srcset="https://state-of-the-art.org/graphics/wand/wand-32-2x.png 2x" style="vertical-align:baseline"/>

It is a common problem to convert one type into another. As with most problems, it is best to begin with the simplest design possible:

    public interface Converter<From, To> extends Repeater
    {
        To convert(From from);
    }

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/convert/convert.svg" width="40"/>

In this design, the *convert()* method converts the *From* type into the *To* type. The interface extends *Repeater* so that any warnings or problems that occur during conversion are captured and [*broadcast to interested listeners*]({ % post_url 2021-07-07-broadcaster % }). 

While this interface perfectly captures a one way conversion, we may want to convert the destination type back to the original type:

    public interface TwoWayConverter<From, To> extends Converter<From, To>
    {
        From unconvert(To to);
    }

Now, a *StringConverter*, as we normally think of it, is just a two-way converter between *String*s and *Value* types:

    public interface StringConverter<Value> extends TwoWayConverter<String, Value>
    {
    }

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/string/string.svg" width="60"/>

The relationships between the classes in the converter mini-framework discussed above can be seen in this UML diagram:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src = "https://state-of-the-art.org/uml/diagram-data-conversion.svg" width="700"/>

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Implementation 

These are fairly elegant interfaces, but what about the implementation?

We would like the base implementation of *Converter* to handle issues that come up that are common to all type converters, namely *null* values and exceptions. We can make use of the [*Polymorphic final methods*]({ % post_url 2021-07-13-polymorphic-final-methods % }) pattern to layer in this functionality for all converters. We will also need to ensure that the converter has at least one listener to hear broadcast problems by making *Listener* a parameter to the constructor.

    public abstract class BaseConverter<From, To> extends BaseRepeater 
        implements Converter<From, To>
    {
        /** True if this converter allows null values */
        private boolean allowNull;
    
        protected BaseConverter(Listener listener)
        {
            listener.listenTo(this);
        }
    
        public BaseConverter<From, To> allowNull(boolean allowNull)
        {
            this.allowNull = allowNull;
            return this;
        }
    
        public boolean allowsNull()
        {
            return allowNull;
        }
    
        /**
         * Converts from the &lt;From&gt; type to the &lt;To&gt; type. 
         * If the from value is null and the converter allows null values, 
         * null will be returned. If the value is null and the converter 
         * does not allow null values a problem will be broadcast. Any 
         * exceptions that occur during conversion are caught and broadcast 
         * as problems.
         */
        @Override
        public To convert(From from)
        {
            // If the value is null,
            if (from == null)
            {
                // and we don't allow conversion of null values,
                if (!allowsNull())
                {
                    // then broadcast a problem
                    problem("${class}: Cannot convert null value", getClass());
                }
                
                // and return null.
                return null;
            }
    
            try
            {
                // Return the converted value.
                return onConvert(from);
            }
            catch (Exception e)
            {
                // If an exception occurs, broadcast a problem
                problem(e, "${class}: Cannot convert ${debug}", getClass(), from);
    
                // and return null.
                return null;
            }
        }
    
        /**
         * The method to override to provide the actual conversion
         */
        protected abstract To onConvert(From value);
    }

In this implementation, any illegal *null* values or exceptions are handled by broadcasting a *Problem* message and returning *null*. This allows converter implementations to focus on the conversion task alone.

Now we can implement our *StringConverter* interface. Besides *null* values and exceptions, *String*s can also be *empty*. To handle this case, we can layer in logic in our base class in a way similar to the way that we dealt with *null* values in *BaseConverter*:

    public abstract class BaseStringConverter<Value> extends BaseConverter<String, Value>
         implements StringConverter<Value>
    {
        /** True if empty strings may be converted */
        private boolean allowEmpty;
    
        protected BaseStringConverter(Listener listener)
        {
            super(listener);
        }
    
        public BaseStringConverter<Value> allowEmpty(boolean allowEmpty)
        {
            this.allowEmpty = allowEmpty;
            return this;
        }
    
        public boolean allowsEmpty()
        {
            return allowEmpty;
        }
    
        @Override
        public final Value onConvert(String string)
        {
            // If we allow null values and our string is null,
            if (allowsNull() && string == null)
            {
                // then return null.
                return null;
            }
    
            // If we allow empty strings and our string is empty,
            if (allowEmpty && Strings.isEmpty(string))
            {
                // then return null.
                return null;
            }
    
            // Return the value of our string converted by the subclass.
            return onToObject(string);
        }
    
        @Override
        public final String unconvert(Value value)
        {
            // If the value is null
            if (value == null)
            {
                // and we allow null values
                if (allowsNull())
                {
                    // return the string representation of null.
                    return nullString();
                }
                else
                {
                    // otherwise, report that we can't convert null values.
                    problem("${class}: Cannot convert null value", getClass());
                    return null;
                }
            }
    
            try
            {
                // Call the subclass to convert the value to a string,
                return onToString(value);
            }
            catch (final Exception e)
            {
                // and broadcast any exception thrown as a problem
                problem(e, "${class}: Cannot unconvert ${debug}", getClass(), value);
                return null;
            }
        }
    
        /**
         * @return The string representation for a null value. 
         * By default this value is null, not "null".
         */
        protected String nullString()
        {
            return null;
        }
    
        /**
         * Implemented by subclass to convert the given string to a value. The 
         * subclass implementation will never be called in cases where 
         * value is null or empty, so it need not check for either case.
         *
         * @param value The (guaranteed non-null, non-empty) value to convert
         * @return The converted value
         */
        protected abstract Value onToValue(String value);
    
        /**
         * Convert the given value to a string
         *
         * @param value The (guaranteed non-null, non-empty) value
         * @return A string which is by default value.toString() if this method is not overridden
         */
        protected String onToString(Value value)
        {
            return value.toString();
        }
    } 

This base class now handles exceptions, *null* values and empty values, only calling *onToString()* with *non-null* values and only calling *onToValue()* with *non-null*, *non-empty* strings. This greatly simplifies the implementation of a converter:

    public class IntegerConverter extends BaseStringConverter<Integer>
    {
        public IntegerConverter(Listener listener)
        {
            super(listener);
        }
    
        protected Integer onToValue(String value)
        {
            return Integer.parseInt(value);
        }
    }

This converter converts from *String* to *Integer* and from *Integer* to *String*. It also handles all boundary conditions: *null* values, *null* strings, empty strings and exceptions, including *NumberFormatException*. All error reporting is via message broadcasting to the listener passed to the constructor. This means that our converter integrates well with the rest of the KivaKit framework. Since *Logger*s are *Listener*s,  we can hook our converter up directly to a logger, and anything that goes wrong will be logged:

    private static Logger LOGGER = LoggerFactory.newLogger();
    
    [...]
    
    var aliens = new IntegerConverter(LOGGER).convert("15");
    if (aliens != null) 
    {
        [...]
    }

We could also capture the errors with *MessageList* and analyze them, or count them with *MutableCount*. Finally, if the class we're working on is a *Repeater*, we can omit the logger declaration and substitute *this*, making our object's error reporting more flexible:

    public MyClass extends BaseRepeater
    {
        [...]
        
        var aliens = new IntegerConverter(this).convert("15");
        if (aliens != null) 
        {
            [...]
        }
        
        [...]
    }

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Conclusion

This article has given a basic description of how KivaKit converters are designed and implemented. In the [*KivaKit command line parsing*]({% post_url 2021-08-19-command-line %}) article, we take a look at how string converters are used to parse command line switches and arguments.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

The simplified conversion code we've explored above is available for use in *kivakit-kernel* in the [KivaKit](https://www.kivakit.org) project.

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
  issue-term="converters"
  theme="github-dark"
  crossorigin="anonymous"
></script>
