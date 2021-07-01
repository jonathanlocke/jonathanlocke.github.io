#### <img src="https://state-of-the-art.org/kivakit-32.png" srcset="https://state-of-the-art.org/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.27 - Functional properties](#with-x)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "with-x"></a>

2021.07.27

### Functional properties &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears-32.png" srcset="https://state-of-the-art.org/graphics/gears/gears-32-2x.png 2x" style="vertical-align:baseline"/>

When objects have properties in Java, this is often expressed with *getters* and *setters*. While [getters and setters should be avoided as much as possible](https://www.infoworld.com/article/2073723/why-getter-and-setter-methods-are-evil.html), some objects need to be configured to be used. We would like their fields to be immutable after construction, avoiding the risks associated with setters. This can be accomplished with constructor(s) or with the [builder pattern](https://dzone.com/articles/design-patterns-the-builder-pattern). But there is another approach that is sometimes useful. We can refer to this pattern as the *functional properties* pattern (since I'm not aware of any existing name for this design pattern). Functional properties preserve immutability while allowing state to be effectively "changed" after construction. This is accomplished by making a *copy* of an object with its state changed accordingly. The use of this pattern looks like this:

    var color = Color.RED
        .withAlpha(128)
        .darker(Percent.of(15));

In this example, the *withAlpha(int)* method is called on the (immutable) *Color.RED* value. This method returns a *copy* of the *RED* object, with the *alpha* value set to 128, but it leaves the original *RED* object unchanged. Next, calling the *darker()* method on this new semi-transparent red *Color* object returns a copy of *that* with reduced brightness:

    red -> semi-transparent red -> 15% darker, semi-transparent red

This pattern can be implemented using a copy constructor, but we will also add a *copy()* method that can be inherited by subclasses (more on the reason for this below).

    public class Color
    {
        public static Color RED = create().withRed(255);
        
        private int red, green, blue, alpha;
    
        public static Color create()
        {
            return new Color();
        }
        
        protected Color()
        {
        }
        
        protected Color(Color that)
        {
            this.red = that.red;
            this.green = that.green;
            this.blue = that.blue;
            this.alpha = that.alpha;
        }
        
        protected Color copy()
        {
            return new Color(this);
        }
        
        public Color withAlpha(int alpha)
        {
            var copy = copy();
            copy.alpha = alpha;
            return copy;
        }
        
        public Color withRed(int red)
        {
            var copy = copy();
            copy.red = red;
            return copy;
        }
        
        public Color darker(final Percent percent)
        {
            final var copy = copy();
            final var scaleFactor = 1.0 - percent.asZeroToOne();
            copy.red = Math.max((int) (red() * scaleFactor), 0);
            copy.green = Math.max((int) (green() * scaleFactor), 0);
            copy.blue = Math.max((int) (blue() * scaleFactor), 0);
            return copy;
        }
         
        [...]
    }

Here, the *create()* method is the only way to create a *Color*, since both constructors are *protected*. In the *red()* static method, once a default color object is created, the *withRed(255)* call makes a copy of that object by calling *copy()*. It then changes the *red* field of the copy to the given value (255) and returns it. The *darker()* method works in a similar way. It makes a copy of its object with *copy()*, computes a new, darker color and returns the copy.

But what if we want to subclass our object? 

It turns out that aliens have x-ray rods and cones in their eyes, so to them colors are red, green, blue, x-ray and alpha. Our alien overlords require an *AlienColor* that is derived from the class *Color*. We would like to reuse the code from the superclass as much as possible, yet add a new value *xray*. We can do this by adding a *withXRay()* method, and then overriding the superclass constructors, the copy method and the *withX()* methods:

    public class AlienColor extends Color
    {
        public static Color create()
        {
            return new Color();
        }
        
        private int xray;

        @Override
        protected AlienColor()
        {
        }
                
        @Override
        protected AlienColor(Color that)
        {
            super(this);
            this.xray = that.xray;
        }
        
        @Override
        public AlienColor copy()
        {
            return new AlienColor(this);
        }
        
        public AlienColor withXRay(int xray)
        {
            var copy = copy();
            copy.xray = xray;
            return copy();
        }
        
        @Override
        public AlienColor withAlpha(int alpha)
        {
            return (AlienColor) super.withAlpha(alpha);
        }
        
        @Override
        public AlienColor withRed(int red)
        {
            return (AlienColor) super.withRed(red);
        }
        
        [...]
    }
    
This is quite a bit of "boilerplate", but in many cases the effort is worth it because our resulting *Color* and *AlienColor* objects have very safe and pleasant APIs. It is easy to discover and use the functional properties of these objects and chain them together. One particularly nice advantage over builders is that you can very easily derive a new object from an existing object. With a builder, you have to build every object from scratch.

You might be wondering why we created the *withAlpha()* and *withRed()* methods that just call the superclass method of the same name? The reason is that they down-cast the superclass *Color* object to an AlienColor object. The flow of control when invoking *withRed()* on an *AlienColor* goes like this:

    AlienColor.withRed() -> Color.withRed() -> AlienColor.copy() -> Color.withRed() -> AlienColor.withRed()

The overridden *copy* method here ensures that if we change the red property of an alien color, the xray property will be preserved. The downcast on *AlienColor.withRed()* returns an *AlienColor* rather than a *Color*. This makes it possible to make a subsequent call to *withXRay()*, like this:

    var color = AlienColor.create().withRed(255).withXRay(128);

The *functional property* design pattern is used frequently in [KivaKit](https://www.kivakit.org).
