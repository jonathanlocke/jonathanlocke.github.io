
#### <img src="https://state-of-the-art.org/kivakit-32.png" srcset="https://state-of-the-art.org/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.13 - The Broadcaster / Listener design pattern](#broadcaster)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "broadcaster"></a>

2021.07.13

### Why the Broadcaster / Listener design pattern is crucial to design

I'm speculating here, but Dr. Alan Kay's conception of object-oriented programming in the early 1960's might have come about, at least partly, as a result of his undergraduate work in molecular biology. A cell is a pretty good analogy for an object and DNA is a sort of template or class from which cells are created. But to Kay, what would have been the most interesting is cell surface receptors and the way that cells pass various kinds of messages to each other (and themselves) by secreting compounds that bind to these receptors. In fact, he posted this revealing email in 1998:

> [...] Smalltalk is not only NOT its syntax or the class library, it is not even about classes. I'm sorry that I long ago coined the term "objects" for this topic because it gets many people to focus on the lesser idea. The big idea is "messaging" [...] 

This brings us to the Broadcaster / Listener design pattern. A language like Java, with statically bound, synchronously invoked methods, is not at all what Dr. Kay is referring to when he talks about messaging. We can't redesign Java as a dynamic messaging-oriented language, but we can do *some* messaging in Java nonetheless. The Broadcaster / Listener design pattern is one way to implement messaging, and it turns out to be very useful and powerful.

A *broadcaster* transmits *messages* to an *audience* of one or more listeners*:

<img src="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener-500.png" srcset="https://state-of-the-art.org/graphics/broadcaster-listener/broadcaster-listener-500-2x.png 2x" style="vertical-align:middle"/>

The

The Broadcaster / Listener design pattern in *kivakit-kernel* is used throughout [KivaKit](https://www.kivakit.org) for messaging between components.



