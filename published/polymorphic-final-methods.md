2021.07.13

### Polymorphic final methods &nbsp; <img src="https://state-of-the-art.org/graphics/override/override.svg" width="60" style="vertical-align:baseline"/>

A *polymorphic* class literally has "many shapes". Subclasses can add functionality and specialize methods of the parent class by *overriding* them:

    public abstract class FlyingObject
    {
        public abstract void launch();
    }
    
    public class Spaceship extends FlyingObject
    {
        @Override
        public void launch()
        {
            countdown();
            blastOff();
        }
    
        [...]
    }
    
    public class JetFighter extends FlyingObject
    {
        @Override
        public void launch()
        {
            taxi();
            waitForTower();
            takeOff();
        }
    
        [...]
    }

In this case, *Spaceship* and *JetFighter* are both *FlyingObject*s, and both implement the *launch()* method in different ways. The *Spaceship* undergoes a countdown and then blasts off. The *JetFighter* taxis to the end of the runway, waits for clearance from the tower and then takes off. 

But all launches require a systems check and a flight status report. How can we provide *all* launches with some common functionality? A simple variation on the [*template method design pattern*](https://en.wikipedia.org/wiki/Template_method_pattern) will do the trick. In this pattern, a method in the base class provides centralized logic which cannot be overridden. This logic calls a method of the same name, but with "on" as a prefix. 

For example, the *launch()* method below is *final*, and it calls *onLaunch()* in the middle of its logic. The *onLaunch()* method is a *protected abstract* method that *can* be overridden, allowing subclasses to indirectly specialize the behavior of the method *launch()* even though it is *final* and cannot be overridden. We can call the *launch()* method below a *polymorphic final method*:

    public abstract class FlyingObject extends BaseRepeater
    {
        public final void launch()
        {
            systemsCheck();
            onLaunch();
            reportFlightStatus();
        }
        
        [...]
        
        protected abstract void onLaunch();
    }
    
    public class Spaceship extends FlyingObject
    {
        @Override
        protected void onLaunch()
        {
            countdown();
            blastOff();
        }
    
        [...]
    }
    
    public class JetFighter extends FlyingObject
    {
        @Override
        protected void onLaunch()
        {
            taxi();
            waitForTower();
            takeOff();
        }
    
        [...]
    }

By providing the ability to "wrap" the *onLaunch()* polymorphic method in logic, we achieve a special kind of leverage. We can affect all implementations of *onLaunch()* by altering the logic in *launch()*. For example, above we do a *systemsCheck()* that is common to all *FlyingObjects* before calling *onLaunch()*, and we *reportFlightStatus()* after the launch completes. If we discover that we want to know how long *any* launch takes we can simply add this logic to the launch method:

    public final void launch()
    {
        systemsCheck();
        var start = Time.now();
        onLaunch();
        information("Launch procedure took $", start.elapsedSince());
        reportFlightStatus();
    }

The *polymorphic final method* pattern is used throughout [KivaKit](https://www.kivakit.org).

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="polymorphic-final-methods"
  theme="github-dark"
  crossorigin="anonymous"
></script>