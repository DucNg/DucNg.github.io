---
title: "Managing a local music library"
date: 2022-11-18T17:22:15+01:00
draft: true
---

Growing up as a kid I mostly listened music through my nice mp3 player.
![old mp3 player](/mp3.jpg)
It created some kind a habit to store music locally and even after streaming platforms were widely used, having limited 3G on my smartphone I mostly relied on playing music stored locally. Over the years, my library grew quiet significantly and became harder and harder to organize. I could at any moment choose the easy pass to just throw everything away and switch everything to a streaming platform but I felt kind of nostalgic and as a tech enthusiast I couldn't just rely on someone elses platform for my needs. So I choose the hard pass to try a have a nice pipeline to store, organize and "deploy" my music.

# Storing music

Of course like everyone I get my digital music from CDs that I rip using [Extract Audio Copy](https://www.exactaudiocopy.org/) 😉

Once ripped it's stored on my [RAID5 NAS](https://ducng.github.io/posts/raid5/). This allows me to access my library from any device that has a network access.

# Organize music

I used to manually put albums I ripped into it's artist folder. It can look innocent at first, but it quickly gets annoying after some time.

Looking for a command line [ID3](https://en.wikipedia.org/wiki/ID3) editor, I came upon an incredibly powerful tool: [beets](https://beets.readthedocs.io).

It changed the vision I had about my library: from something organic, manually crafted, to something that can get incredibly organized and automated.

# "Deploy" music

* Plex
* beet convert
* web js browse https://github.com/Nomeji/caddy-music
