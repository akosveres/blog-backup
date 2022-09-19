## XTRadio architecture

I wanted to write a blog post about [XTRadio](https://xtradio.org) for a very long time. I finally have the time, so let's start with a short but sweet post which explains how everything works, how it all fits together.

## What is XTRadio

XTRadio is an online radio station which started on 08.08.08. It's a joint venture between my good friend `crayon` and myself. It started on the Undernet IRC Servers, `Silvered` was heavily involved as well. It's not the biggest project you'll ever hear about, for us involved, it holds a special place in our hearts. A post detailing the history of the radio should happen in the future!

We play electronic music 24/7.

## Architecture and diagram



![xtradio.drawio(1).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663598640969/e4UWi6w5Y.png align="center")

In the above diagram you can see the full picture, we'll dive deeper into this, let's have two categories:
* Audio route
* Data route

## Audio

We play music, dooh! We are not streaming from other websites, we have good old mp3 files. Why not OGG or OPUS? In 2008 it was just easy to set it all up with mp3 and we just stuck to it.

We need to upload, edit and store these MP3 files somehow. In the good old days we just ran an FTP server, upload the files, have the backend read it and we were done. Over time we wanted to more data with each MP3 file, like proper `Artist`, `Title`, `Source URL`, etc.

### Admin interface

Enter the Admin interface.

It's a very simple web application, it went through a couple of iterations and even now it's not ready for prime time, but essentially it's used for the following actions:

* Upload
* List
* Edit details
* Create playlist file

The most interesting aspect is the file upload. Once you select a file from your computer and you upload it, you have a form which allows you to change the Title, Artist, Source URL. An automatic search to Spotify and Soundcloud is made so we can get the album part for the song. The album art is uploaded to the CDN, stored on disk and a unique file name is returned.

All this information is stored in a MySQL database. We need it later, especially on the website.

The uploaded MP3 file is stored in a folder which is shared with Liquidsoap.

### Liquidsoap

This piece of software is a beast! Super fast, extensible, reliable. Check out more details on [liquidsoap.info](https://liquidsoap.info).

In a nutshell, we grab a bunch of mp3 files and play them with Liquidsoap.

We can do metadata manipulation, encoding, decoding, call APIs on certain events, save stream output, a telnet interface to manipulate song queues, etc.

The first time the Admin interface spins up, it generates a playlist file based on the entries in the mysql database. Here's a few lines from the `playlist.txt` file:
```
annotate:artist="Brandy",title="Full Moon (Final DJs Remix)",length="335",image="5004360098.jpg",share="https://soundcloud.com/finaldjs/brandy-full-moon-final-djs":/MUSIC/BACKUP/remix/Brandy_-_Full_Moon_(Final_DJs_Remix).mp3
annotate:artist="London Grammar",title="Wasting My Young Years (Sound Remedy Remix)",length="404",image="6161727760.jpg",share="https://soundcloud.com/soundremedy/wasting-my-young-years-sound":/MUSIC/BACKUP/remix/London_Grammar_-_Wasting_My_Young_Years_(Sound_Remedy_Remix).mp3
annotate:artist="The Black Keys",title="Little Black Submarines (Ageless Remix)",length="302",image="973019753.jpg",share="https://soundcloud.com/ageless84/little-black-submarines":/MUSIC/BACKUP/remix/The_Black_Keys_-_Little_Black_Submarines_(Ageless_Remix).mp3
annotate:artist="Andrew Gentry Remix",title="Poetic Justice (ft. Clams Casino) Remix",length="228",image="3840532694.jpg",share="https://soundcloud.com/andrewgentrymusic/poetic-justice-ft-casino-clams":/MUSIC/BACKUP/remix/Poetic_Justice_(ft._Clams_Casino)_Remix.mp3
annotate:artist="Kyla La Grange",title="Cut Your Teeth (Kygo Remix)",length="396",image="9869146691.jpg",share="https://soundcloud.com/kygo/kyla-la-grange-cut-your":/MUSIC/BACKUP/remix/Cut_Your_Teeth_(Kygo_Remix).mp3
```

We have the following information in here:
* Artist
* Title
* Song length
* Image unique file name on the CDN
* Source URL
* File location

Liquidsoap uses each of this information in various ways, the length is the most important for it, it knows for sure how long a song is, this way it can determine when to auto-queue the next file.

We decode the file we randomly pick from the playlist, grab all the information we can, re-encode the file into 3 bitrates, start playing the file and send an API call to our REST API.

When you connect to listen to the radio, you connect straight to [Icecast](https://icecast.org), you have a choice of various bitrates:
* AAC 32kbps
* MP3 128kbps
* MP3 320 kbps

Liquidsoap re-encodes our MP3 files and sends it to the icecast endpoints.

You can find our Liquidsoap script [on github](https://github.com/xtradio/xtradio-liquidsoap/blob/master/radio.liq).

## Data

Having an audio stream is awesome and everything, but on a website you can't just have a a player and that's it. Not that our website is super detailed, but it's not just an audio player.

Our data is available through a public API, anyone can view it [here](https://api.xtradio.org/api), (the v2 is available [here](https://api.xtradio.org/v1/np/)).

The source of the data is from our MySQL database and Liquidsoap. When a song plays, liquidsoap sends an event to the API, this prompts a request to the DB to request the details about the song. In the v2 API we're also getting the upcoming songs, straight from the Liquidsoap telnet interface, details of this code is [here](https://github.com/xtradio/xtradio-api/blob/master/telnet.go).

Based on this we can put together the details on the website, in the end it looks like this

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663600364555/LAdfoXVzm.png align="center")

The background color is always random, determined based on the album art.

## Fin

If anyone is interested in more details, feel free to reach out and we can discuss other aspects. The next article in this series should discuss the infrastructure aspects, how we deployed everything, how we used Civo cloud and kubernetes.

All details about xtradio are available on github on [github.com/xtradio](https://github.com/xtradio).

