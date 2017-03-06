---
title: using ffmpeg to demux stream(video h.264 , audio aac) and do the time search
date: 2017-03-06 20:38:57
tags: ffmpeg
---

[ffmpeg](http://ffmpeg.org/) is a very famouse multimedia library.

##There are several sites I think are very helpful for multimedia developer:##

1   [ffmpeg detail](http://3xin2yi.info/wwwroot/tech/doku.php/tech:multimedia:ffmpeg)

2   [ffmpeg decode procedure](http://www.douban.com/note/228831821/)

3   [An ffmpeg and SDL Tutorial](http://dranger.com/ffmpeg/)


##Today I want to talk how to demux network stream by ffmpeg, the full procedure is as following:

1   register movie container format and codec by calling av\_rigister\_all()

2   open the network stream by calling 'av\_open\_input\_stream()'

3   find the stream info by calling 'av\_find\_stream\_info()'

4   loop all the stream and find audio and video by checking the 'codec\_type' of each 'stream(nb\_streams)'

5   alloc memory for packet of frame by calling 'av\_init\_packet' or 'avcodec\_alloc\_frame'

6   read frame from stream by calling 'av\_read\_frame'

7   check the type of stream(audio or video)

8   after read all the frame calling 'av\_close\_input\_stream' to close the stream

##if want to do time search,the procedure is as following:

1   call 'av\_seek\_frame' to seek by time or by byte

2   call 'av\_read\_frame' to read packet and do demux and decode

#refer source code:

1   [ffmpeg demux/remux](https://github.com/joyxu/joyxu.github.com/tree/master/attachment)

2   [ffmpeg demux](https://github.com/joyxu/joyxu.github.com/tree/master/attachment)
