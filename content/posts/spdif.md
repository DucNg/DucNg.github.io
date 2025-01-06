---
title: "Using 5.1 through S/PDIF (toslink) on Linux"
date: 2020-10-20T18:07:42+02:00
---

Using 5.1 on Linux isn't something very common in the first place. But using it over the S/PDIF (toslink) is even less common so finding documentation
about how to do it is close to impossible. After hours and hours of struggling, I found a working solution so I decided to write an article about it.

## What is 5.1?

The 5.1 technology allows audio to be sent to the speaker through 6 independant channels: 
left, right, center, rear left, rear right and the .1 for the subwoofer (the bass).

Most setups only use 2 channels. This is called 2.0 or simply stereo.

We use 5.1 to create immersion and reproduce the cinema experience.

## What is S/PDIF?

S/PDIF is an old connector (the 90') that uses optical fibers to transmit digital audio. It sounds cool but in practice it isn't at all.
It supports sending stereo audio as PCM and 5.1 using Dolby Digital (ac3) or DTS (dca) compression. So it only supports lossy audio for 5.1.

PCM is the most basic way to transmit audio. It's just a soundwave: a digital representation of an analog signal.

This connector is obsolete and has been replaced by HDMI that supports sending audio through up to 8 independants PCM channels.

Why use S/PDIF then? Unfortunatly my amplifier (home cinema) only supports 5.1 through S/PDIF. So it's just a matter of not buying a new one that supports HDMI (150€ for a used one).

## What are Dolby Digital (DD) and DTS?

Dolby Digital and DTS are compression methods used to send 5.1 audio using only 2 channels. The basic concept is to use different frequencies for each individual channel.
Of course both are proprietary and you should have a licence to use them. Both compression and decompression: your computer needs a licence, your amplifier also needs one.

Both are lossy audio compression methods. They have been replaced by DolbyTrueHD (DTHD) and DTS-HD (used in bluray) that support lossless 5.1.

There are a lot of other compressors that support 5.1. Even though you could create a file with 6 PCM channels, the most common is to use whatever
format was used in the source bluray (DTS-HD or DTHD). But using 6 FLAC channels or even 6 AAC channels isn't that uncommon either.

## How audio works on Linux?

I'm not going to go into much detail here. Just the basic most common setup.

![linux audio schema](/pcm.svg)
**Audio on Linux using default setup**

The user is playing audio with a media player or any application. The audio file can be in any format so it's usually decompressed before being played.
Here, decompression means converting from any audio format to PCM. This PCM audio is then sent to PulseAudio that mixes the different audio
sources and then sends that to ALSA. ALSA then sends the audio through the cables (jack, HDMI, DP, S/PDIF).

With this setup you basically only get stereo PCM audio to the speaker. It's the most common thing to do and it works for most use cases.

## How to get 5.1?

tl;dr: use this https://github.com/darealshinji/dcaenc

By default PulseAudio supports sending PCM 5.1 through HDMI. It's easy to do: the player decompresses every channel to PCM from any given format, then PulseAudio
takes that and plays it.

But the problem lies here: my amplifier cannot understand PCM 5.1 and moreover there isn't even enough bandwidth on the S/PDIF cable.

If you search on the internet on how to do it, what most people recommend is to use passthrough. That's what Kodi recommends for example.
The concept of passthrough is to send audio from the player to ALSA without decompressing the audio on the way. The goal is to send the audio
file to the cable (digital audio cable: HDMI, DP or S/PDIF) without any alteration on the way. The decompression job is then left to the amplifier.

So the thing here is to assume that the amplifier has the correct decompression codec built-in to decode the audio file. It's kind of wierd but it's 
really how it works. Modern amplifiers have DTS-HD and DTHD decompressors built-in that allow them to play bluray audio losslessly. Even for 
my old amplifier it works in the same way, it has a licence and built-in decompressor for DTS and DD.

So the idea here is to configure the player to use passthrough. In kodi it's just a checkbox. But anyways, it never worked for me.
No matter what file I used, my amplifier only showed the "PCM" icon, I don't know if PulseAudio decompressed on the way or whatever happend.

The thing is, this method isn't ideal. You have to have a file that your amplifier will support. I don't think any amplifier will ever
support 5.1 AAC for example. It could be the reason it didn't worked for me, maybe I didn't have any suitable file.

Another problem is that we would need to bypass PulseAudio because it cannot mix a encoded sources. But
PulseAudio already focus the only possible audio source.

So I could have stopped my journey here and say "there is no way I will have audio compressed in the right format sent directly to the cable".
But in fact there was a way. The a52 plugin.

## a52 ALSA plugin

The main idea is to compress audio on the ALSA side before sending it to the cable. So, you decompress the audio software side then send it to 
PulseAudio in PCM. PulseAudio send sthis mixed PCM to ALSA, ALSA compresses it to DD and sends this to the audio cable. It's as simple as that.

The thing is, it's so niche that it seems like nobody ever did that on the internet. I even asked on the ALSA channel on Freenode but it seem nobody
has ever heard of the a52 plugin or didn't even use an ALSA plugin in the first place.

But the solution was there. a52 comes with the "alsa-plugins" package and it was just a matter of how to make it work.
The problem is, the only documentation you can find is here: https://github.com/Themaister/alsa-plugins-rsound/blob/master/doc/a52.txt
It's basically the only documentation on the subject. But no matter how hard I tried, I never got it to work. For example the documentation
doesn't say how to make this work with PulseAudio.

The problem could also be licencing. Of course, you need a licence to compress audio to DD. 
This page: https://www.alsa-project.org/main/index.php/A52_plugin seems to imply that the plugin isn't included by default for licence reasons and 
you have to recompile alsa-plugins yourself to get it. The thing is, there is absolutely no documentation on how to compile it and enable 
the a52 plugin on the way.

Since I couldn't get it to work, I searched for an other solution, the last possible one. Convert PCM to DTS.
And a solution for this exists: https://github.com/darealshinji/dcaenc

It's an external ALSA plugin that does exactly that. The main difference, there is a complete documentation from A to Z on how to make it work,
even covering the PulseAudio part. It's a standalone plugin so it's easier to compile. 
Once configured in PulseAudio there is a new audio source called Digital Audio DTS Transcoding (HDMI/DP); 
in reality it's the S/PDIF output, the naming is just wrong. Once selected, the amplifier instantly switched to DTS.

Then from any audio source it is converted to DTS. Stereo is played as stereo in DTS format, 5.1 is played as real 5.1 also in DTS format. 
My problem was solved for real.

![linux audio dca schema](/dca.svg)
**Audio on Linux using dcaenc**

## Bonus: the idea for the solution

One of my main problems during this whole project was to understand how it could work. The system is complex and has many different ways
to work and not work. The idea for the "convert everything" solution came from my PlayStation 3. The game console was able to use my 5.1
perfectly well, without any additional configuration and at first I couldn't understand why. I came to realize that Sony had the licence
to convert any source to DTS and DD, so they used that to make everything work. The only thing was to understand how to do this on Linux.

