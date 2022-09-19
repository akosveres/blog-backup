## Stream Icecast Stream To Youtube

At XTRadio we thought of taking our stream to a new platform (or even more), the first obvious choice was Youtube Live. Facebook Live could be another option but it only allows 4 continous hours of stream, so it wouldn’t allow us to stream our 24/7 broadcast.

Youtube Live has been easy enough to setup, we’ve used ffmpeg on a Debian server, it takes an image and our stream, puts them together and it pushes to Youtube. Easy peasy, eh? One thing to keep in ming: make the image HD, then the stream will be HD as well, of course use the best stream you have. Here’s our way of running things:

```
avconv -loop 1 -i /MUSIC/xt-hd.png -i http://localhost:8080/mp3 -r 30 -acodec aac -strict -2 -c:v libx264 -strict experimental -b:a 128k -pix_fmt yuvj444p -b:v 256k -minrate 128k -maxrate 512k -bufsize 768k -f flv rtmp://a.rtmp.youtube.com/live2/[KEY]
```

(code is still WIP)

You can listen to the XTRadio Youtube Live Stream if you please, the next step is to bake this in to Liquidsoap and have the image display the song information as well, eventually have some sort of live equalizer. And run all of this dockerized, but that’s another post.