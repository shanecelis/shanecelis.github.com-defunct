---
layout: post
title: "Google Summer of Code 2013: Emacsy"
description: ""
category: Emacsy
tags: [gsoc, emacsy, emacs, guile, vim]
---
{% include JB/setup %}

Emacsy was accepted as a [2013 Google Summer of Code
Project](http://www.google-melange.com/gsoc/homepage/google/gsoc2013),
which is great news!  Emacsy is an embeddable Emacs-like library for
[GNU Guile](http://www.gnu.org/software/guile/) Scheme. It's an
attempt to extract "the Emacs way" of using, augmenting, and extending
an application while it is running. Emacsy harbors no ambition to
become a text editor because there is already a great Emacsy text
editor: Emacs.  I'd like to use this post to address some features and
implementation details that I'm personally excited about: how to do
online help, going beyond Emacs to do job control, and some comments
on whether Emacsy's "shell" ought to be special.

The [GSoC proposal has many
details](https://google-melange.appspot.com/gsoc/proposal/review/google/gsoc2013/shanecelis/1)
about the project, but perhaps I should say who this blog entry is
written for, as I see a handful distinct audiences: application users,
integrators, and contributors.

* Application users are the users of some application X that Emacsy
  has been integrated into it.  As Emacsy hasn't been released, there
  is no real application users to speak of yet, but they're important
  to keep in mind.

* Integrators are developers who embed Emacsy into their own
  application.  They need to know enough to integrate with it but
  don't necessarily care about how Emacsy does its thing.

* Contributors are developers who are interested in the architecture
  and inner-workings of Emacsy and can contribute to its development.

These Emacsy blog posts will be of most interest to would-be contributors and
integrators, and I will assume some familiarity with Emacs.  There's a
tension between features that might be nice to have but would incur a
large burden on integrators.  It'll be helpful to keep that tension in
mind when deciding what features to implement and how.


## Help will always be given in Emacsy to those who ask for it

Emacs has the most comprehensive online help system of any
application; there is really nothing else like it. This is the reason
for its claim to be
[self-documenting](http://www.emacswiki.org/emacs/SelfDocumentation).
Unfortunately because there isn't anything else like it, it seems a
bit alien at first.  If you don't know what the key sequence `M-C-\`
does, then hit `C-h k M-C-\` and Emacs will tell you that it's bound
to the procedure
[`indent-region`](http://www.gnu.org/software/emacs/manual/html_node/elisp/Region-Indent.html).
(If this notation `C-h` for control-h is unfamiliar, please see [Sacha
Chua's excellent one pager for
beginners](http://sachachua.com/blog/2013/05/how-to-learn-emacs-a-hand-drawn-one-pager-for-beginners/).)
If you don't know what the procedure `indent-region` does, then hit
`C-h f indent-region` and Emacs will describe it and even provide a
link to the source code.  One question I've been struggling with is,
how can Emacsy provide online help like Emacs does? Several ideas come
to mind:

* **Text buffer, duh.** One idea is to duplicate the way Emacs does it
  by showing a read only text buffer. In Emacs all things are text
  buffers, but Emacsy does not make this presumption. Also if one must
  display a text buffer that means every integrator must figure out
  how to display this text. What if the view is larger than the
  window? Scrollbar, please. What if there are links in the text?
  Should they be clickable? Of course! Integrator, please implement
  these text features that may have nothing to do with your
  application.  This seems like too high a burden for integrators,
  even if it would be cool.

* **Minibuffer to the max.** Just use the minibuffer for
  everything. We can get away with using the minibuffer for a lot of
  things but that would not allow us to have a comprehensive online
  help system that was comparable to Emacs'.

* **Use the Web, Luke.** This idea struck me recently. What if when we
  wanted to show a lot of information, say about some particular
  function `C-h f some-fun`, we opened a link to some internal web
  server? Emacsy serves up the information over HTTP and the browser
  gives us a lot of features like links, styling, rich media, and
  perhaps some customization possibilities. The implementor only needs
  to send URL requests to some handler application.  Fancier
  implementors could handle these URLs internally in a web view.

I'm pretty taken with this last idea.  It retains the ability to let
the application be fully self-documenting without imposing a heavy
burden on the implementor.


## Going Beyond Emacs: Job Control

One feature I'm particularly excited about is job control. How many
times have you accidentally stumbled upon some network operation that
causes Emacs to freeze? Granted it'll come back if you wait, or you
can quit `C-g` it. But wouldn't it be nice to just suspend that
command like you would in the shell?  Just hit `C-z` to suspend
it. Then maybe type `M-x background` to continue running it in the
background. That's a feature that we can implement in Emacsy, but how?
Allowing for job control brings up the issue of multitasking.

### Multitask like it's 1999

How shall Emacsy support running multiple jobs some of which are in
the background?  I'm going to suggest a [**cooperative
multitasking**](http://en.wikipedia.org/wiki/Computer_multitasking#Cooperative_multitasking.2Ftime-sharing)
approach rather than a [**pre-emptive
multitasking**](http://en.wikipedia.org/wiki/Computer_multitasking#Preemptive_multitasking.2Ftime-sharing)
approach.  Heresy, I know, and I know it's not the '90s, but hear me
out.

I like to think of Emacsy as inheriting from both Emacs and Unix.
Unix and Emacs are somewhat odd bedfellows though. Unix is about small
tools that do one thing well and compose easily; the shared state is
the filesystem with little context, and it's generally not focused on
interactive apps. Emacs on the other hand is about having all tools,
big and small, at one's finger tips (key bindings); the shared state
is Emacs' process memory with lots of context, and it's focused on
interactive use.

For an operating system like Unix pre-emptive multitasking is
absolutely preferred.  Note though that the memory of each process is
isolated from one another, so you don't have to worry about one
process overwriting another's memory.  However, an Emacsy command will
not have its memory isolated from other commands, so it seems
inadvisable to use a pre-emptive multitasking mechanism like threads
since the commands could then overwrite shared memory unless very
carefully synchronized. Instead I suggest using coroutines to
implement a cooperative multitasking system.  Only one piece of Emacsy
code runs at any one time; thus a whole class of race conditions
evaporate.  This doesn't preclude one from using threads within Emacsy
since threads are natively supported by GNU Guile, but they will have
to work out the synchronization details themselves.

## Why not vimy?

There are two text editors of note: Emacs and vim, so why Emacsy and
not vimy?  Partly because I couldn't do a vimy project justice because
I'm not a vim user, but I think there is a larger reason than that. I
think vim is an excellent text editor. Some of my friends are absolute
wizards with it. I've had glimers of understanding the [power and
unity of its terse
sub-language](http://stackoverflow.com/questions/1218390/what-is-your-most-productive-shortcut-with-vim?page=1&tab=votes#tab-top). I
am envious of the dot `.` command. However, making the best text
editor, sub-language and all, does not necessarily mean it would
transfer well to other domains. Is an insert mode necessary in a CAD
program?  What is the terse sub-language of manipulating chemical
models? I think making something worthy of the name vimy would be very
challenging in another domain.

That said, I don't see any reason why someone couldn't write a
vim-like UI using Emacsy as a starting place.  Borrowing more from the
Unix tradition, consider the shell.  The creators of Unix didn't make
the shell special.  It is just another program unlike DOS' shell, but
that required a lot of ingenuity to invent the right six system calls
to make that happen (Torvalds, ["Just for Fun"
54](http://books.google.com/books?id=--K-DvEj7yAC&lpg=PA54&ots=UnNf_jdHxV&pg=PA54#v=onepage&q&f=false)).
In [my
proposal](https://google-melange.appspot.com/gsoc/proposal/review/google/gsoc2013/shanecelis/1),
I emphasized the Key-Lookup-Execute-Command-Loop (KLECL), but in a way
that is essentially Emacs' shell.  One could write a different shell,
a vim-like shell if they preferred.  That is, if Emacsy takes care to
not make the shell special.

# That's it for now

Those are some of my current musings for Emacsy.  I'm going to try to
record further ideas on [Emacsy's github
wiki](https://github.com/shanecelis/emacsy/wiki).  Feel free to
contribute your own ideas there or write a comment below.  Otherwise,
I'll be on [twitter](https://twitter.com/shanecelis) or hanging out in
`#guile` on `freenode.net`.
