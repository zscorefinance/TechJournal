Distribution of streaming media is best done over HLS or MPEG-DASH. Modern browsers are equipped to handle media streams simply using HTML5 `<video />` with media source extensions and can play HLS seamlessly. Interestingly almost all video infrastructure we know today are powered by FFmpeg. HLS works on HTTP/S (which is just TCP underneath). The media delivery backend could simply be an nginx server, configured to distribute steaming media over HTTP.

For example, I downloaded a clip from YouTube in 720p mp4 format and renamed it to "cwc11_final.mp4", and executed the following command:

```bash
ffmpeg.exe 
 -i "cwc11_final.mp4" 
 -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 
 -c:v libx264 
 -crf 22 
 -c:a aac 
 -ar 44100 
 -filter:v:0 scale=w=480:h=360 -maxrate:v:0 600k -b:a:0 500k 
 -filter:v:1 scale=w=640:h=480 -maxrate:v:1 1500k -b:a:1 1000k 
 -filter:v:2 scale=w=1280:h=720 -maxrate:v:2 3000k -b:a:2 2000k 
 -var_stream_map "v:0,a:0,name:360p v:1,a:1,name:480p v:2,a:2,name:720p" 
 -preset fast 
 -hls_list_size 10 
 -threads 0 
 -f hls 
 -hls_time 3 
 -hls_flags independent_segments 
 -master_pl_name "cwc11_final.m3u8" 
 -y "cwc11_final-%v.m3u8"
```

The command created a master playlist file `cwc11_final.m3u8` and three other media playlist files, `cwc11_final-360p.m3u8`, `cwc11_final-480p.m3u8` and `cwc11_final-720p.m3u8`; along with multiple time segment files (as instructed in the command options), for each resolution type specified. These small files are tiny segments of the media, if played in sequence, one after another, will play the entire media as it is.

The m3u8 files are like index files or metadata files for the media. The master playlist file `cwc11_final.m3u8` contains:

```bash
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-STREAM-INF:BANDWIDTH=1210000,RESOLUTION=480x360,CODECS="avc1.640015,mp4a.40.2"
cwc11_final-360p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2232120,RESOLUTION=640x480,CODECS="avc1.64001e,mp4a.40.2"
cwc11_final-480p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=3882120,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
cwc11_final-720p.m3u8
```

The master playlist file contains details about the other media playlist files in various resolutions available. When a client (browser) asks for this media, the server responds with the master playlist file. This is when the media source extensions in the browser gets to know about all the available resolutions and corresponding bit rates, which the browser will adapt to, depending on the network speed available on the client device. Higher the bandwidth, the browser will use the 720p m3u8 playlist file to fetch the 720p media segments and lower the bandwidth the browser may fall back to 240p. This is known as adaptive steaming over HTTP. The tiny media segments will be delivered to the client ahead of time and played in sequence. 

<img src="./assets/live-streaming-part-2_2.jpeg" width="100%"> <br />
<img src="./assets/live-streaming-part-2_3.jpeg" width="50%"> <br />