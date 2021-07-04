
#### <img src="https://state-of-the-art.org/graphics/speech/speech-32.png" srcset="https://state-of-the-art.org/graphics/speech/speech-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.08 - Mirror methods](#mirror-methods)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "mirror-methods"></a>

2021.07.08

### Mirror methods &nbsp; <img src="https://state-of-the-art.org/graphics/mirror/mirror-32.png" srcset="https://state-of-the-art.org/graphics/mirror/mirror-32-2x.png 2x" style="vertical-align:baseline"/>

This bite-sized article describes a simple syntactic idea that could make object-oriented programming a little nicer. One legitimate criticism of object-oriented programming is that it sometimes pretty arbitrary which object should have a method. 

For example, since aliens are able to convert themselves to and from pure energy, we might write:

    public class Alien
    {
        public PureEnergy toPureEnergy()
        {
            [...]
        }
    }
    
    public class PureEnergy
    {
        public PureEnergy toAlien()
        {
            [...]
        }
    }

But we could just as easily write:

    public class Alien
    {
        public static Alien fromPureEnergy(PureEnergy energy)
        {
            [...]
        }
    }
    
    public class PureEnergy
    {
        public static PureEnergy fromAlien(Alien alien)
        {
            [...]
        }
    }

The code in *Alien* seems preferable since it doesn't use static methods, but what if the conversion between aliens and pure energy involves a lot of calculations? We may want to keep those calculations private, since the implementation may change. There might also be code that we could share between the two conversions (alien to pure energy and pure energy to alien). And we will want to keep that code private to improve encapsulation.

One way to manage this might be to create a converter interface like:

    public class AlienTransmogrifier
    {
        Alien fromPureEnergy(PureEnergy energy);
        PureEnergy toPureEnergy(Alien alien);
    }

The logic is separate now and more flexible, but we have a whole extra class that is going to have to access properties of *both* classes to perform calculations (breaking encapsulation). And in some cases, maybe we *don't* want this flexibility, so the class is just a waste.

This is where the mirror methods come in. What if we could define all of our energy conversion logic in one class, say *Alien*, but make it accessible in *PureEnergy*, without writing any additional code?  

First, lets put all of our extraterrestrial physics logic in one place, in *Alien*:

    public class Alien
    {
        public PureEnergy toPureEnergy()
        {
            [...]
        }
        
        public static Alien fromPureEnergy(PureEnergy energy)
        {
            [...]
        }
    }

This is great for encapsulating our conversion logic. But now if we have a *PureEnergy* object we can't just say *energy.toAlien()*. We won't even know that this conversion is possible unless we happen to look at *Alien*. That's not good for API ease-of-use.

We can solve this whole problem by adding an annotation (and plenty of [*handwaving*](https://en.wikipedia.org/wiki/Hand-waving)):
 
    public class Alien
    {
        @MirrorMethod("toAlien")
        public static Alien fromPureEnergy(PureEnergy energy)
        {
            [...]
        }
    }

The *@MirrorMethod* annotation here specifies that the *PureEnergy* class should automatically (without writing any code) have the static method *PureEnergy.toAlien()*:

    public class PureEnergy
    {
        public Alien toAlien()
        {
            return Alien.fromPureEnergy(this);
        }
    }

The same goes for the other mirrored method. The *toPureEnergy()* method:

    public class Alien
    {
        @MirrorMethod("fromAlien")
        public PureEnergy toPureEnergy()
        {
            [...]
        }
    }

would automatically imply the existence of this method:

    public class PureEnergy
    {
        public static PureEnergy fromAlien(Alien alien)
        {
            return alien.toPureEnergy();
        }
    }

Now, we have centralized, encapsulated logic that supports all four of our conversion use cases and is easily discovered with IDE autocompletion. 

    alien.toPureEnergy()
    energy.toAlien()
    PureEnergy.fromAlien(Alien)
    Alien.fromPureEnergy(PureEnergy)

We did this by simply adding two *@MirrorMethod* annotations.


