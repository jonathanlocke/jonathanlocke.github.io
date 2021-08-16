2021.07.05

### Mirror methods &nbsp; <img src="https://state-of-the-art.org/graphics/mirror/mirror-32.png" srcset="https://state-of-the-art.org/graphics/mirror/mirror-32-2x.png 2x" style="vertical-align:baseline"/>

This article describes a simple syntactic idea that could make object-oriented programming a little nicer. One legitimate criticism of object-oriented programming is that it is sometimes pretty arbitrary which object should have a method. 

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
        public Alien toAlien()
        {
            [...]
        }
    }

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/alien/alien.svg" width="55" style="vertical-align:middle"/> 

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

The code in the first example seems preferable since it doesn't use static methods, but  our code is split between two classes. What if the conversion between aliens and pure energy involves common code that we might like to keep private to a single class?

One way to get around this might be to create a converter interface like:

    public class AlienTransmogrifier
    {
        Alien fromPureEnergy(PureEnergy energy);
        PureEnergy toPureEnergy(Alien alien);
    }

The logic is separate now and more flexible, but we have a whole extra class that is going to have to access properties of *both* classes to perform calculations (possibly breaking encapsulation). And in some cases, maybe we *don't* want this flexibility, so the class is just a waste.

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

This is where mirror methods come in. What if we could define all of our energy conversion logic (it turns out to be quite easy) in one class, say *Alien*, but make it accessible in *PureEnergy*, without writing any additional code?  

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

Now, the idea. What if we could solve this whole problem by just adding an annotation? With plenty of [handwaving](https://en.wikipedia.org/wiki/Hand-waving) concerning compilers and language specs, we have:
 
    public class Alien
    {
        @MirrorMethod("toAlien")
        public static Alien fromPureEnergy(PureEnergy energy)
        {
            return [...]
        }
    }

The *@MirrorMethod* annotation here specifies that the *PureEnergy* class should automatically (without writing any code) have the mirror method *PureEnergy.toAlien()*:

    public class PureEnergy
    {
        public Alien toAlien()
        {
            return Alien.fromPureEnergy(this);
        }
    }

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

The same goes for the other mirrored method. The *toPureEnergy()* method:

    public class Alien
    {
        @MirrorMethod("fromAlien")
        public PureEnergy toPureEnergy()
        {
            return [...]
        }
    }

would automatically imply the existence of this mirror method:

    public class PureEnergy
    {
        public static PureEnergy fromAlien(Alien alien)
        {
            return alien.toPureEnergy();
        }
    }

Now, we have centralized, encapsulated logic in the *Alien* class that supports all four of our conversion use cases and is easily discovered with IDE autocompletion: 

    alien.toPureEnergy()
    alien.fromPureEnergy(PureEnergy)
    energy.toAlien()
    PureEnergy.fromAlien(Alien)

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

*But how does the mirror method feature figure out how and where to implement these methods?*. 

All mirror methods follow these rules:

1. The mirror method has to have the same return value
2. The implementation of a mirror method has to call the method being mirrored
3. If the mirrored method is static with no parameters, it cannot be mirrored
4. If the mirrored method is static with one or more parameters, the mirror method will be an instance method in the class of the mirrored method's first parameter. It will pass *this* as the first parameter when calling the mirrored method.
5. If the mirrored method is an instance method with no parameters, the mirror method will be a static method in the class of the return value. It will take the enclosing class of the mirrored method as the class of its first parameter and will use that reference to call the mirrored method.
6. If the mirrored method is an instance method with one or more parameters, the mirror method will be placed in the class of the first parameter. It will take the enclosing class of the mirrored method as the class of its first parameter, and it will pass *this* as the first parameter when calling the mirrored method.
7. Any parameters beyond the first parameter will be parameters to the mirror method and will be passed through when calling the mirrored method.

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

So, the mirror method of this (repeated from above):

    public class Alien
    {
        @MirrorMethod("toAlien")
        public static Alien fromPureEnergy(PureEnergy energy)
        {
            return [...]
        }
    }
        
has to look like this (by rules 1, 2 and 4):

> (4) If the mirrored method is static with one or more parameters, the mirror method will be an instance method in the class of the first parameter. It will pass *this* as the first parameter when calling the mirrored method.

    public class PureEnergy
    {
        public Alien toAlien()
        {
            return Alien.fromPureEnergy(this);
        }
    }

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

In the case of the other mirror method above, the mirror method of this:

    public class Alien
    {
        @MirrorMethod("fromAlien")
        public PureEnergy toPureEnergy()
        {
            return [...]
        }
    }
    
will be this (by rules 1, 2 and 5):

> (5) If the mirrored method is an instance method with no parameters, the mirror method will be a static method in the class of the return value. It will take the enclosing class of the mirrored method as the class of its first parameter and will use that reference to call the mirrored method.

    public class PureEnergy
    {
        public static PureEnergy fromAlien(Alien alien)
        {
            return alien.toPureEnergy();
        }
    }

<img src="https://www.kivakit.org/images/horizontal-line-128.png" srcset="https://www.kivakit.org/images/horizontal-line-128-2x.png 2x" />

Let's take a look at another example using the same process:

This mirror method here:

    public class Distance
    {
        @MirrorMethod("speedToGo")
        public Speed per(Duration duration)
        {
            return [...]
        }
    }    

infers the existence of this method (by rules 1, 2 and 6):

> (6) If the mirrored method is an instance method with one or more parameters, the mirror method will be placed in the class of the first parameter. It will take the enclosing class of the mirrored method as the class of its first parameter, and it will pass *this* as the first parameter when calling the mirrored method.

    public class Duration
    {
        public Speed speedToGo(Distance distance)
        {
            return distance.per(this);
        }
    }
    
Now we can say either of these two things:

    var speed = Distance.miles(1).per(Duration.minutes(4));
    var speed = Duration.minutes(4).speedToGo(Distance.miles(1));

and both resolve to the same code.

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="mirror-methods"
  theme="github-dark"
  crossorigin="anonymous"
></script>
