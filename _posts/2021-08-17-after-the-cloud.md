2021.08.17

### What's after the cloud? &nbsp; <img src="https://state-of-the-art.org/graphics/chips/chips.svg" width="40" style="vertical-align:baseline"/>

At present, the software world is dominated by work on "Big Data" and "AI" that takes place in "The Cloud". Looking back at the era of mainframes in the 1950's, it must have been exceptionally difficult to see microcomputers, cell phones and the Internet coming, or that mainframes would be obsoleted and left behind, only existing to run legacy software. So it is with the cloud today. We are at a unique point in the evolution of computing machines, and something is next. But what?

In Thomas Kuhn's seminal book on the sociology of science, *The Structure of Scientific Revolutions*, scientists work within a well-defined and accepted paradigm. During such periods of "normal science", scientists make discoveries that are useful and exciting, but not especially surprising or upsetting. They do day-to-day work to advance the state of the art in a steady, incremental fashion. 

Within a paradigm, one of the primary functions of science is to offer models that accurately predict observations. If the model fails to accurately predict the outcome we observe, there are three possibilities:

1. Experimental error
2. The model can be made to work with some refinement(s)
3. The model can no longer work, and a new model is required

When evidence starts to arise that can't be accounted for within the current model, the scientific community strongly prefers options 1 and 2. Using or developing a new model might be possible, but the community begins to resist through: 

1. Denial of anomalous evidence
2. Assertion of authority
3. Strong criticism of new ideas
4. Insistence that experimental error must be the cause
5. Proposals for workarounds and refinements to reduce predictive errors
6. Attacking other scientists

The history of science is littered with ruined careers and missed opportunities as a result of group resistance to better models. Science is hardly a rational enterprise in the short term. But in the long term it wins out, or at least as far as we know it does.

It is only when a new and better model arises with the capacity to explain anomalous observations that the new model becomes adopted. The workarounds start to become harder to use than the new model. This shift, of course, forms a new paradigm to work within, and the start of a new period of normal science.

It's interesting that Kuhn made these observations of science but not of technology or other human pursuits. Technology also has a sociological structure, and we see a similar pattern: (1) periods of "normal technology", followed by (2) resistance and (3) the emergence of a new technological model forming a new period of normal technology.

The pace of this is especially fast in computation. In terms of the evolution of electronic computational systems, the progression to date is something (only very roughly) like this:

| Decade | New Technologies                                                     |
|--------|----------------------------------------------------------------------|
| 1880's | Relays, logic gates                                                  |
| 1890's | Hollerith tabulator                                                  |
| 1930's | Special-purpose computers                                            |
| 1940's | Memory, transistors, von Neumann architecture                        |
| 1950's | Mainframes                                                           |
| 1960's | Microprocessors, graphical interfaces                                |
| 1970's | Minicomputers, UNIX, ethernet, microcomputers, distributed computing |
| 1980's | Personal computers, AI                                               |
| 1990's | Cell phones, Internet                                                |
| 2000's | Cloud computing, big data                                            |
| 2010's | Quantum computers                                                    |

At present, The Cloud is our era of "normal technology" (quantum computers exist, but nobody seems to know quite what to do with them). So, given that major shifts have occurred in virtually every decade in the past, what can we expect is coming next?

One likely possibility is that a prior paradigm is upset due to its inability to function as a model (just as in Kuhn's work). In particular, two problems are holding back computing at present:

1. Heat dissipation
2. Density of computation and storage

It's possible that a small shift like Gallium Arsenide (GaAs) chips can extend the current computational paradigm. This might give us faster chips (THz, not GHz) and less heat output. However, it wouldn't represent an actual paradigm shift. In particular, it wouldn't give us anything qualitatively different from computation in the current era under The Cloud. 

So what *would* be new?

Perhaps the von Neumann architecture spanning from the 1940's to the present day will be the next to go. We are already seeing evidence of the need to work around von Neumann design in modern chips and vector processors. But how could the whole paradigm shift?

One possibility is "molecular computing", where the density of computation and storage might increase radically. The internet may seem like a big thing. It's estimated that it is of the order of a zettabyte (10^21 bytes or 10^22 bits). This is an interesting number because it's very close to 10^23, which is the order of magnitude of a mole (of any substance, a mole of water weighing about 18 grams). So, if we were able to store data in terms of water molecules, where each molecule stored a bit, we could store 10^23 bits, or 10 present-day internets worth of bits. In 18 grams.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <img src="https://state-of-the-art.org/graphics/nucleus/nucleus.svg" width="80"/>

It will take time for computing to evolve in this direction, but it surely will. And along with it will go the von Neumann model. Molecular computers will need to be qualitatively different from von Neumann machines. They will have giant amounts of memory, but they will likely have to perform computations more locally, as a centralized bus and a clock pulse won't scale to this size. Whatever the actual form of molecular computers, they will (probably temporarily) obviate our present need for The Cloud. So what will we have instead? Some kind of localized computing again.

The driver for this paradigm shift will be the same as the last major paradigm shifts: 

> Why would you want your massive pile of sensitive, proprietary data under someone else's control on an Internet cloud like AWS, when you can put it on one or two devices that are under your full control? If your big data division can meet all its needs with a small number of machines, why would you even connect those machines to the internet?

Of course, a decade later, we will probably have applications so demanding that a few odd applications (maybe computational biology) will require that molecular computers be put into a "Molecular Cloud". 

The transitions that have occurred between paradigms in computation have more to do with ownership and control of computation and data than with technology itself:

| Years            | Era                           |
|------------------|-------------------------------|
| 1880's to 1950's | Mainframe era (Centralized)   |
| 1960's to 1980's | Microchip era (Decentralized) |
| 1990's to 2020's | Internet era (Centralized)    |
| Unknown          | Molecular era (Decentralized) |

The decentralization that occurred during the microchip era had to do with demand from managers and executives who wanted full control of the systems that they used (in the 1950's IBM mainframes were rentals leased from and maintained by IBM). Will the same occur in reaction to the cloud era? Is AWS the new IBM, renting computation to companies that will ultimately want control of their computing resources back? The only thing stopping companies from going back to desktop computers is that they can't handle the required computational load. If molecular computers of not-too-distant future can do what entire cloud systems do today, why would anyone use the cloud except to run legacy software?

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="cloud"
  theme="github-dark"
  crossorigin="anonymous"
></script>
