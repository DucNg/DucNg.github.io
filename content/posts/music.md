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

The concept is pretty simple, put all your unorganized library in a folder, run `beet import .` in this folder and get a fully organized music folder in return. beets uses existing tags on your files and match it with music databases like [MusicBrainz](https://musicbrainz.org/) or [Discogs](https://www.discogs.com/). Using those data, it will fix and complete the tags of your files, put them in the correct directory and rename them. It does this according to the configuration you have defined. Here is mine as an example:

```yml
directory: /home/duc/Music

paths:
    default: $albumartist/[%if{$original_year,$original_year,$year}] $album%aunique{}/$track. $title
    comp: Compilations/[%if{$original_year,$original_year,$year}] $album%aunique{}/$track. $title

import:
    incremental: yes
```

The `directory` option allows me to set the directory where I want my organized library to be. This is not easily modifiable in the future since this path will be used to identify file location in the beets database. We don't need to specify the folder for unorganized music since it doesn't really matter, you can run the `beet import` command on any folder.

The `paths` option define how your files will be organized. My way of organizing music isn't too fancy but it gives you a glimpse of how deep you can go with beets.

First of, let's explain why there is 2 paths defined here. Default is for the usual stuff, usually albums with a single artist (eventually with featuring), compilation on the other hand, is for stuff that compile music from different artists so it would not make sense to store them in an artist folder. The comp path is defined by beets but you can also add custom ones using queries if you want. For example, if you can't remember soundtrack composers very well and you would prefer using album names directly you could do that (as seen on the beets documentation):

```yml
paths:
    albumtype:soundtrack: Soundtracks/$album/$track $title
```

This will store music tagged as "soundtrack" in a Soundtrack directory with the name of the album.

Back to my configuration. First variable is `$albumartist`, this is pretty straightforward this is just the name of the artist who created the album. Don't confuse it with `$artist` that indicates the name of the artist(s) on a specific track. This usually includes featuring for example. beets takes really good care to not mess up these two.

Something I like to see albums ordered by years of release even when seen from a file browser, so I added a `[$year]` before the album name. But this is not enough! Sometimes, albums are re released into better versions or recording but I usually have a single version in my library so I don't really care about that. That's why I have 2 variables set: `$original_year` the original year of the release of the album in case the album is a re release and `$year` for the usual case where this is just a simple first release of an album. beets allows me to say "use this one if available" using a simple if condition.

Then comes the `$album%aunique{}` thing. I won't go too deep on this one since it's already explained [in the documentation](https://beets.readthedocs.io/en/stable/reference/pathformat.html#aunique). Just think of it as the name of the album without having to worry about duplicates.

And to finish, `$track. $title`, `$track` is for the track number and `$title` is for the title of the track. An interesting anecdote about that is, the first time I did my configuration, I forgot to add this `$track` thing and ended up with tracks sorted in alphabetic order. Realizing it I immediately added it, ran `beet move` and BAM, in a few seconds all my library was organized the way I wanted. This is when I realized how powerful beets could be.

Another cool thing about track numbers is that it doesn't takes into account multiple CDs. What I usually had before beets is 2 separate folders namely "Cool album CD1" and "Cool album CD2" and tracks ordered from 1 in each. I usually hated that since it forces you to go back to the UI to run the second CD after the first one and it's ugly to have twice the same album shown in the album tab of you player when it fact it's the same. With beets, tracks are ordered in natural order even if on a second CD. If each CD contains 5 tracks, first track of the second CD will just be named 6. But beets still keep the info of which tracks belongs to which album and store it into the ID3. So if you player support it (like mines) you'll get a beautiful CD separators between tracks but everything on the same page. Like so:

![example of multiple CDs in Plex](/plex.webp)



# "Deploy" music

* Plex
* beet convert
* web js browse https://github.com/Nomeji/caddy-music

# It is not perfect yet

More than one album a year?
