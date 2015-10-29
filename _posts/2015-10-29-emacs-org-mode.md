---
layout: post
title: Getting Started with Emacs Org-Mode
---

A long time ago, in a galaxy far, far away, I was an emacs user. We had an internal version of emacs written
by Warren Montgomery. It was small, and fast. It didn't have the vi mode issues that drove me crazy in vi.
I loved it. Warren's emacs didn't have things like elisp, but it was good for editing C code, writing nroff
documents, and all the miscellaneous things in text files. Around two years ago, I got interested in
clojure. I was trying to format a PDF booklet for printing, and it contained a lot of complex tables. I was
usinng Ruby and Prawn. Unfortunately, prawn wasn't up to the task and the fix looked hard. Since I was working
on a deadline, I needed to look for a different solution, and I had heard about clojure and had some lisp background.
I discovered that I could do most of what I needed with clj-pdf, and I could extend it a little to do the rest.
I found that I really liked clojure, so I joined [The Den of Clojure](http://www.meetup.com/denofclojure/) . I rediscovered emacs, at least a little.

Now, I really like the cursive IntelliJ plugin for clojure, and I also like the JetBrains Ruby plugin and
RubyMine product, but I have been intrigued with how well people can do pretty much everything without leaving
emacs. Also, everyone needs a plain text editor sometimes. IntelliJ is not a great way to deal with the random
text file that is not part of an IntelliJ project that needs a quick tweak.

I will admit that I have been frustrated learning emacs. The Gnu emacs keybindings are not the same as the ones
I remembered, and it is a much more complex program. I was able to move forward a little bit at a Den of Clojure
Office hours dedicated to emacs, and I joined the [Denver Emacs Meetup](http://www.meetup.com/Denver-Emacs-Meetup/) to learn more.

This month, Lee Hinman did a great demo on org-mode. Since it can generate markdown, I decided to try it for a
few posts. The first one had some structure and a fair amount of code. For the most part, it wasn't too hard.
Recent versions of emacs include org-mode by default, and there is a really [complete document](http://orgmode.org/org.html) describing org-mode
in gory detail.

I ran into a couple of problems. The first was handling of links. Org-mode links are inserted using
[[link-path][text] ], but as soon as you type it, the file shows a link. When I wanted to edit one, it took me
a bit of trial and error to realize that just hitting delete would remove a bracket and then then I could edit the
information in the link. The second problem was leaning how to enable markdown export. The documentation said markdown
export wasn't enabled by default, it took a bit of searching to find out that I needed to add

    (eval-after-load "org"
        '(require 'ox-md nil t))

to my init.el to export markdown.

I haven't tried the todo or notes capabilities of org-mode, but after doing a blog post with a bunch of source code,
I can see how it can be used to create literate programming mixes of source code and documentation. The features to
edit and execute code fragments are really helpful.

At this point, I can't say that I will switch to emacs for everything, but I am learning enough to be comfortable using
emacs for more tasks.
