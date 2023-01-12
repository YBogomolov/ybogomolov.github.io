---
layout: post
title: "With Simplicity Comes Clarity"
author: yuriy
categories: [ essay, knowledge management, personal, zettelkasten ]
image: assets/images/simplicity-clarity.jpg
---

I decided that I want to write not only about programming, but also about other things that deeply interest me — like education, knowledge management, or leadership methods. In this short essay, I describe my journey in knowledge management and what system I use in my daily work.

<!--more-->

{% note %}
With this short essay I want to begin a new page in my writing, challenging myself to write not only about programming but also about things that deeply interest me — education, knowledge management, leadership, team management, and so on. Hope you find these articles as useful and insightful as my writings about functional programming.
{% endnote %}

For years I was trying different methodologies of personal productivity, chasing the goal of becoming smarter and more efficient: David Allen's *[Getting Things Done](https://gettingthingsdone.com)*, [its variation](http://www.time-mngmnt.narod.ru) by Vasiliy Kislyi, *[Jedi Techniques](https://mnogosdelal.ru)* by Maxim Dorofeev, etc., etc., etc. All these methods at first felt like *the* solution, giving me a sense of boosted productivity and effectiveness. But after some time I unavoidably arrived at the conclusion that they bring nothing more than added complexity into my life. Only a few of them made their path into my habits, really, with [Inbox Zero](https://www.43folders.com/43-folders-series-inbox-zero) being the most prominent example.

When I reflect on this topic, I think this inevitably happens when the methodology is not simple and/or automated enough. My thoughts are quick, and a spark of an idea needs to be written quickly, as it may be replaced with others almost immediately. This leads to ideas being lost, which in its turn brings to the table irritation with the sluggishness of the instrument. To add more, when I wrote the idea down, I had to write it in a specific form — say, put a few tags, or decide in which folder to put it, or something else. It felt slow and unproductive: I had to focus on the *infrastructure* rather than on the *content*.

Eventually, I realised that what I really need is a *knowledge management* system, and began researching this area. Almost immediately I stumbled upon [Zettelkasten](http://zettelkasten.de) — a system of intralinked notes serving as a second brain. My good friend recommended that I should try [Obsidian](https://obsidian.md), and I completely fell in love with this software. Another journey began — this time, in search of a perfect Zettelkasten workflow. For quite some time I tried to shove myself into someone else's workflows — using Zero Links[^1], using a system of tags, using folders to organise notes, etc. Still, nothing felt right — the method stood in my way instead of helping me organise my thoughts and work. 

And then I discovered Andy Matuschak's *Evergreen notes*.

If you haven't heard of him, Andy Matuschak is an engineer and researcher who worked at Apple and Khan Academy, and whose main topic of research is knowledge management. His notes [are available for public access](http://notes.andymatuschak.org), and they serve as a great example of a working Zettelkasten-esque method. Personally, I find it very amusing just to browse his intraconnected network of notes, and in particular, this is joyful thanks to his very clear academic style of writing.

Before I describe the core idea, I will name the things I always wanted from a knowledge management system:
1. a place to dump "raw" ideas, notes, sketches, drawings, etc. — in short term, some kind of inbox;
2. minimal to nonexistent structure — as a programmer and architect, I already deal with increased cognitive load, so I don't want to spend my time thinking where the heck should I put this particular idea and which tags to assign; I want to *be* productive, not *struggle* doing so;
3. availability on every platform — on my Mac and on my iPhone, predominantly;
4. not tied to a particular proprietary format, and, more importantly, not being a subscription-based SaaS *(looking at you, Evernote)* — I want to be the only owner of my second brain and decide whether *I* want to sync it to a cloud or not, where to store it, how to do backups, etc.

A combination of Obsidian and Evergreen notes tick most of these boxes:
- ✔️ Obsidian uses Markdown to store notes. Markdown files are just text, so I can edit my knowledge base anywhere and with any editor — Obsidian, VSCode, Vim, TextEdit, and so on. This not only liberates my mind by lifting the need to worry about closed-source proprietary storage format but also makes such notes easily publishable in my blog using GitHub Pages.
- ✔️ Obsidian vaults are just folders, so they could be stored anywhere, including clouds. Right now I store my main vault in a cloud folder of one of the major cloud providers, but with a snap of the fingers can also just dump it on the USB drive and forget about network connectivity.
- ✔️ Obsidian is really snappy and available on mobile and desktop platforms. It's not that convenient on the small screen, but I don't think this really matters, as I use my iPhone to either work on the ready notes or dump momentary ideas into my Inbox.
- ✔️ I found out that Daily notes are the perfect Inbox. I incorporated an overview and cleanup of my daily notes into my morning routine, trying to stick to the Inbox Zero approach. This allows me to quickly dump everything that bothers me *somewhere*, and know that it won't go under the radar unnoticed because I will re-read and re-think this idea during an overview.
- ✔️ Simplicity of how Andy Matuschak keeps his notes gave me inspiration and understanding that for knowledge the best structure is an emergent structure — i.e., a structure that appears on its own during the lifetime of the system. I dropped all those tags, zero links, templates and whatnot, and this felt like a breath of fresh air. Simplicity and minimalism bring calm to my mind and allow me to focus on my own ideas and thoughts.

Right now, my Vault contains only three folders — `Base` for the core notes, `Files` for attachments like images and PDFs[^2], and `Inbox` for holding temporary daily notes and other things that weren't processed yet and made their path to the core network. I decided that I won't be using any artificial ways of organising notes, thus allowing an emergent structure to appear. I don't think I can go simpler than that.

I am quite happy with what I learned — it was a bumpy road, but it gave me valuable experience and a deeper understanding of myself. The journey is not over yet, but the first time I feel confident in my knowledge management system.

[^1]: a special set of pages prefixed with `0` or `00` that serve as hubs for other notes
[^2]: There's a subtle benefit of having files separated from notes: it's easier to do an emergency backup that should be small enough to be sent by email. Files are important, too, but the notes are what really matters.
