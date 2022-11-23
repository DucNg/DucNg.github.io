---
title: "Managing a music library using beets"
date: 2022-11-18T17:22:15+01:00
---

Growing up as a kid I mostly listened music through my nice mp3 player:
![old mp3 player](/mp3.jpg)
It created some kind a habit to store music locally and even after streaming platforms were widely used, having limited 3G on my smartphone I mostly relied on playing music stored locally. Over the years, my library grew quiet significantly and became harder and harder to organize. I could at any moment choose the easy pass to just throw everything away and switch everything to a streaming platform but I felt kind of nostalgic and as a tech enthusiast I couldn't just rely on someone elses platform for my needs. So I choose the hard pass to try a have a nice pipeline to store, organize and "deploy" my music.

# Storing music

Of course like everyone I get my digital music from CDs that I rip using [Extract Audio Copy](https://www.exactaudiocopy.org/) 😉

Once ripped it's stored on my [RAID5 NAS](https://ducng.github.io/posts/raid5/). This allows me to access my library from any device that has a network access.

# Organize music

I used to manually put albums I ripped into it's artist folder. It can look innocent at first, but it quickly gets annoying after some time.

Looking for a command line [ID3](https://en.wikipedia.org/wiki/ID3) editor, I came upon an incredibly powerful tool: [beets](https://beets.readthedocs.io).

It changed the vision I had about my library: from something organic, manually crafted, to something that can get incredibly organized and automated.

The concept is pretty simple, put all your unorganized files in a folder, run `beet import .` in this folder and get a fully organized music folder in return. beets uses existing tags on your files and match it with music databases like [MusicBrainz](https://musicbrainz.org/) or [Discogs](https://www.discogs.com/). Using those data, it will fix and complete the tags of your files, put them in the correct directory and rename them. It does this according to the configuration you have defined. Here is mine as an example:

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

First of, let's explain why there is 2 paths defined here. Default is for the usual stuff, albums with a single artist (eventually with featuring), compilation (comp) on the other hand, is for stuff that compile music from different artists so it would not make sense to store them in an artist folder. The comp path is defined by beets but you can also add custom ones using queries if you want. For example, if you can't remember soundtrack composers very well and you would prefer using album names directly you could do that (as seen on the beets documentation):

```yml
paths:
    albumtype:soundtrack: Soundtracks/$album/$track $title
```

This will store music tagged as "soundtrack" in a Soundtrack directory with the name of the album.

Back to my configuration. First variable is `$albumartist`, this is pretty straightforward this is just the name of the artist who created the album. Don't confuse it with `$artist` that indicates the name of the artist(s) on a specific track. This usually includes featuring for example. beets takes really good care to not mess up these two.

Something I like is to see albums ordered by years of release even when seen from a file browser, so I added a `[$year]` before the album name. But this is not enough! Sometimes, albums are re released into better versions or recording but I usually have a single version in my library so I don't really care about that. That's why I have 2 variables set: `$original_year` the original year of the release of the album in case the album is a re release and `$year` for the usual case where this is just a simple first release of an album. beets allows me to say "use this one if available" using a simple if condition.

Then comes the `$album%aunique{}` thing. I won't go too deep on this one since it's already explained [in the documentation](https://beets.readthedocs.io/en/stable/reference/pathformat.html#aunique). Just think of it as the name of the album without having to worry about duplicates.

And to finish, `$track. $title`, `$track` is for the track number and `$title` is for the title of the track. An interesting anecdote about that is, the first time I did my configuration, I forgot to add this `$track` thing and ended up with tracks sorted in alphabetic order. Realizing it I immediately added it, ran `beet move` and BAM, in a few seconds all my library was organized the way I wanted. This is when I realized how powerful beets could be.

Another cool thing about track numbers is that it doesn't takes into account multiple CDs. What I usually had before beets is 2 separate folders namely "Cool album CD1" and "Cool album CD2" and tracks ordered from 1 in each. I usually hated that since it forces you to go back to the UI to run the second CD after the first one and it's ugly to have twice the same album shown in the album tab of you player when it fact it's the same. With beets, tracks are ordered in natural order even if on a second CD. If each CD contains 5 tracks, first track of the second CD will just be named 6. But beets still keep the info of which tracks belongs to which CD and store it into the ID3. So if you player support it (like mines) you'll get a beautiful CD separators between tracks but everything on the same page. Like so:

![example of multiple CDs in Plex](/plex.webp)

Last option is:

```yml
import:
    incremental: yes
```

This allows me to import new music from my "unorganized music folder" with just one command. I would usually have to remember what was already imported and what wasn't yet and ran several `beet import` for each albums. With this option I can just run `beet import .` and beets will remember what was already imported from this folder and only import the new stuff.

And this covers my import configuration! So to summaries, I now have a command line tool that allows me, with a single command, to import, organize and tag all my music files. Since it's command line based, I run it directly on my NAS from SSH, no need to move files around between computers.

# "Deploy" music

The question is now, how to listen to this perfectly organize music? When I was 13, this question was easily answered, just drag & drop everything onto the MP3 player and you're good to go! There is no way it makes more than 4 Go right?

![music folder size 138G](/folder-size.png)

Yeah, I kinda store my music as flac now and it can get pretty big. There is no way I'm able to store all of it at once on a portable device. So first solution is to access it via network!

## Accessing music via js

It's NAS after all, it's made to have connectivity! My first approach was a funny one. I stole some js code from a music sharing website that allowed playing music from a simple http server index directory. It's just vanilla JavaScript and HTML5 player. I adapted it to work as a [Caddy browse template](https://caddyserver.com/docs/caddyfile/directives/file_server#browse). Unfortunately, I cannot share the code since I stole and it was not open source in the first place, it also doesn't have a license. But I don't think it's an ideal solution anyway. I can still demonstrate how it works tho. [Here is a video of it in action](/musicjs.webm). It really showcase how a well organize music library doesn't require a player scanning all ID3 to be usable.

This worked well for a time and I'm still using it on some occasions. It's very versatile, it can work on any device with an internet access and a web browser. It's also very light. But it lakes some cool functionalities like searching tracks or album across the whole library, playlist and even track list and showing what's playing system wide. So I decided to go for a more complete solution:

## Accessing music via [Plex](https://www.plex.tv/)

Okay so this is still HTML5 + JavaScript but in much more refine package. I won't go too deep on this one since it's a quiet well known software already. Just to say, it works really well as a music player the UI is at least as good as Spotify's one. It's also really easy to setup. It's an ideal solution except it's not open source 🙁. It also works for series and films so it very handful and it works on many devices like phones, smart TVs or gaming console. This is what I use most of the times.

## Accessing music on my phone

So here is the complex part. Accessing music from network works well and I can already do it via Plex but for my phone, I like to keep things offline. Or at least some. Just in case there is no mobile connectivity somewhere.

My first approach to this problem was just to transfer FLAC files from my pc to my phone. I would swap files on some occasion and I could have a nice subset of my library that I carried around (plus it was lossless). But after having a phone without a sdcard slot, I quickly realized this couldn't last. My library kept growing and I kept wanting to listen to many different things at once on my phone. So I took the big decision: 

>It has to end. I will listen to lossy audio now (at least on my phone).

First step was finding the right codec. Quickly looking around, [the only right answer is obviously opus](https://opus-codec.org/comparison/). [At 96 Kb/s](https://wiki.xiph.org/Opus_Recommended_Settings#Recommended_Bitrates) you get an almost "transparent" experience so that's what I went for since I have very limited storage.

Now we need a find a way to get those files converted and transferred. Thankfully beets also has our back on this one. Beets has a plugin called convert that allows to convert files from your library to a specific format. This is what I added to my config file:

```yml
convert:
    dest: /home/duc/Lossy musiques
    never_convert_lossy_files: yes
    embed: yes
    auto_keep: yes
    album_art_maxwidth: 200
    copy_album_art: yes
    format: opus
    formats:
        opus:
            command: ffmpeg -i $source -y -vn -acodec libopus -ab 96k -f opus $dest
            extension: m4a
```

This allows me to run `beet convert QUERY` and have what I want converted into a folder. Don't look too much at the custom ffmpeg command, this is just a workaround to make Samsung read my files.

Running the command manually can be useful when I want to transfer old music to my phone but pretty painful when it's music I've just ripped and imported. That's why [I made a PR](https://github.com/beetbox/beets/pull/4302) to add a new option, the one you can see here: `auto_keep` that auto convert files after they've been imported. It's one interaction less to do!

After being converted, I manually transfer files to my phone via wifi using TotalCommander. It's not perfect but it works well enough.

# It is not perfect yet

As I just said, this far from being perfect, it's much better than it was but it still requires more interactions than I would like.

A nice addition would be the ability to auto synchronize the lossy music folder on my NAS with the music folder on my phone. I tried to do this using rsync but didn't succeeded, no Android rsync app could manage ssh keys. Another solution could be to use [Syncthing](https://syncthing.net/) but I didn't found a way to only have files added and not deleted (making the sync only one way). I think it's doable but it requires more investigation.

I also still need to run `beet import` manually after each rip which is a pain. I tried to automate it using inotify but beets wasn't made to be autonomous, it often requires you to manually take decisions on how you want your music to be tagged. So I gave up.

There are also some issues with beet in itself. Some artist are not correctly tagged on MusicBrainz creating duplicates in your library it can be frustrating and it's not easy to solve. There is also the issue with japanese artists, since I don't read japanese at all, I have trouble remembering how their name is written in japanese so it makes it hard to find them in the library. It's also a problem from command line when you can't type the name of the folder. A nice solution could be to have them written in [romaji](https://en.wikipedia.org/wiki/Romanization_of_Japanese) but [apparently it's no easy task](https://github.com/beetbox/beets/issues/1916).

But overall, I'm still really happy of how it ended up. And of course there is still room for improvement! Don't hesitate to send me an email if you have ideas on how to improve this setup or just to sharing your own!

See yaa~!
