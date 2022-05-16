+++
title = "Time to take a hard look at securing files"
date = 2022-05-16
extra.toc = false
+++

In the past decade or so, loads of research piled into secure
messaging, and, with the likes of the Signal protocol and its
derivatives, the research reached mass adoption. Not only that, but we
have a range of competing implementations that offer different privacy
tradeoffs! [Supergeil!](https://www.youtube.com/watch?v=YyTJYI-JpHU)
	
For some reason, the same effort has not piled into securing
files. While we have [age](https://github.com/FiloSottile/age) gradually
displacing GPG for messaging (see a pattern?), the vast majority of
file security and backup softwares are rooted in the 90s.

In the rest of the article I'm going to rant a bit before I arrive at
*my* solution. [Jump ahead](#so-how-do-we-fix-file-storage) if you're in a
rush.

## Usable security (or lack thereof)

On the one hand, we have robust tools that have been tested and vetted
through exposure to time, which is great! Many eyeballs make every
usability hell shallow (or something like that).

On the other hand, this feels backwards: people are getting top notch
user interfaces from Silicon Valley startups and gigacorps, while they
have to navigate archaic and arcane configurations to secure their
academic research with VeraCrypt.

How should anyone know whether TwoFish is a reasonable choice of
encryption, or they need to mix it with
[TurboFish-GUI-SIV](https://turbo.fish/) mode using [Visual
Basic](https://www.youtube.com/watch?v=hkDD03yeLnU) to make this
secure for their needs?

Proprietary software for file storage is cute, but mostly built for
the enterprise market. And let's not delude ourselves about the
quality of this stuff, these products are typically optimized for
revenue over security.

*"Can I still access my data trivially in 10 years time?"* is a
question I've heard multiple times during my research. Some people
I've spoken to specifically avoid proprietary tools because their
longevity is questionable.

It seems that currently we can't have it both ways: it's either
secure, or usable. 

## And then: portability.

Many of open source backup software are written in Python. Performance
hit the bottle's neck quickly and the fix was to add some C. And maybe
do some `exec()` calls to a few ready-made software that's going to
cut down on code size. Now you need a few different package managers
to handle the complexity.

All of this stuff is unapologetically designed for Linux, with a
typical Linux user in mind from the 2000s when we were all compiling
Gentoo from `stage1` to achieve best in class desktop performance (we
never did).

Recently, I had to spend a week away from my regular desktop, with a
Linux laptop. *"I have backups!"*, or so I thought. Until I realised
it's going to take a non-trivial amount of time to recover those from
encrypted TimeMachine archives. It slowly sank in that my flawless
strategy is utterly useless if I change one variable: the part of the
computing stack that runs a browser and an Emacs, the operating
system.

Try using any Python open source stuff on Windows, and you'll feel the
pain. WSL2 is cheating, that's a [Linux
system](https://www.youtube.com/watch?v=dxIPcbmo1_U)!

## And then: performance.

Needless to say, when any system depends on layers and layers of code
and third party tools, it becomes a lot harder to tame
performance. But things get even worse when you consider networks.

In the past I've made the mistake of blindly shoveling data into a
cloud backup provider with an open source client. I've even tested my
backups! It was great. An endless pit of data on the cheap.

That is, until I had to do a full system restore. Downloading and
unpacking about 1TB took 3 or 4 days on a fast European Internet
connection. That's when I realised that while I had lots of
*bandwidth*, the *latency* to access chunks of my data were
horrifically slow. Welp.

## So how do we fix file storage?

In the last year of the Before Times, I wrote a prototype of
[Zerostash](https://github.com/symmetree-labs/zerostash) in Rust, with
a few design goals in mind:

 * Make it easy to re-implement in arbitrary languages
 * Allow for a large number of use cases without hellish configuration
 * Make it [secure by default](https://github.com/symmetree-labs/infinitree/blob/main/DESIGN.md)
 * Make it fast

The truth is, the original scheme sucked. Not because I messed up the
goals, thankfully. It was an overly complex design that ultimately
scaled badly. Many features were impossible to implement. It was
really, very fast, though.

So about a year ago I resigned at my dayjob, and decided spend my
funemployed year learning how to enjoy writing software again by
making Zerostash not suck.

And here we are. Zerostash is now a fully usable, multiplatform,
incremental, deduplicated backup tool that doesn't suck (so much).

The CLI is modelled vaguely after Git commands, since it works pretty
similarly to Git internally. You get to `commit`, `checkout`, `ls`, and
`log` your archives. 

## Where does a newborn go from here?

One of the aspects of
[Zerostash](https://github.com/symmetree-labs/zerostash) and the
underlying [Infinitree](https://github.com/symmetree-labs/infinitree)
library that I like the most is that the design allows building a
bunch of different user experiences around one secure core.

 * Integrate it silently into a GUI thingy? Sure.
 * File systems through FUSE? Why not.
 * Sync & restore your workspace anywhere at high speeds? Please do.
 * Encrypted Git? Yes please.
 * Transparently encrypt all your customer data in Amazon S3? Hell yeah.
 * Store your microservice application state in the cloud before reload?
   Of course.
 * Build a replicated, encrypted database to replace your SQL logic
   with native Rust iterators? Ehm, maybe...

In the coming weeks, I will go through some design decisions and
solutions that went into Infinitree and Zerostash so they scale up and
down.

There are also a bunch of mundane things to implement, like garbage
collection, or integrating streams into the CLI to allow ZFS piping
snapshots into Zerostash.

## Fuck around and find out

Going forward, I would love to continue working on Zerostash and the
surrounding Rust ecosystem in a focused way, and I am looking for
people who would support this journey!

If you or your organization is in need of improving your data
management story, why not **hire me**?

Send me an email at **p at symmetree.dev**!
