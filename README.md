# Nginx 搭配 RTMP HLS 建立簡易直播 Server

* [Youtube Tutorial - Nginx 搭配 RTMP HLS 建立簡易直播 Server - 概念 -part1](https://youtu.be/YIXgNUrvqp0)

* [Youtube Tutorial - Nginx 搭配 RTMP HLS 建立簡易直播 Server - 實戰 -part2](https://youtu.be/oxnUUlCXpE8)

簡易版架構圖

![alt tag](https://i.imgur.com/gYWMjoX.png)

## 教學

請確認電腦有安裝 `ffplay` 以及 `ffmpeg`

ffplay 用來播放影片 (拉流)

```cmd
ffplay -version
```

FFmpeg 用來將影片(轉檔)送到 server (推流)

```cmd
ffmpeg -version
```

在 [docker-compose.yml](docker-compose.yml) 底下, 所使用的 docker image 是
 [nginx-rtmp](https://hub.docker.com/r/tiangolo/nginx-rtmp/)

(注意, 這個容器底下是沒有 FFmpeg 的)

底下的 [stat.xsl](stat.xsl) 是要看 server 推拉流的狀態.

檔案來源為 [nginx-rtmp-module/blob/master/stat.xsl](https://github.com/arut/nginx-rtmp-module/blob/master/stat.xsl)

[nginx_simple.conf](nginx_simple.conf) 則是簡易版的設定, 底下分別設定了 RTMP 以及 HLS 的參數.

詳細參數說明可參考 [nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module/wiki/Directives)

啟動指令 `docker-compose up -d`

可以先來看 server 推拉流的狀態

[http://your_ip:8088/stat](http://your_ip:8088/stat)

![alt tag](https://i.imgur.com/72TOzYq.png)

## 推拉流

推流

將本地的影片以 stream 的方式推給 Nginx RTMP Server

```cmd
ffmpeg -re -stream_loop -1 -i project_view.mp4 -c:v copy -c:a copy -f flv rtmp://your_ip:1935/live/test
```

test 是你的 stream key, 可以定義自己喜歡的.

![alt tag](https://i.imgur.com/KmlPV0B.png)

指令說明

`-stream_loop number (input)`

`0` 代表不 loop

`-1` 代表無限 loop

`-re` 讀取輸入來源(流)原生的 frame rate

`-i` 輸入來源

`-c:v copy` 等於 `-vcodec copy` 影片格式，copy 代表與來源相同

`-c:a copy` 等於 `-acodec copy` 聲源格式，copy 代表與來源相同

`-f flv` 轉檔成 flv

這邊是用 FFmpeg 來完成推流, 也可以用軟體, 像是最常見的 OBS 推流也是可以的.

拉流

從 Nginx server 拉流 (stream) 下來觀看

`RTMP 拉流`

RTMP (Real-Time Messaging Protocol) 延遲比較低.

(但有可能被防火牆擋下來)

```cmd
ffplay rtmp://your_ip:1935/live/test
```

剛剛設定的 stream key 是 test, 所以這邊也要選擇 test

也可以禁用快取, 會比較沒有延遲

```cmd
ffplay -fflags nobuffer rtmp://your_ip:1935/live/test
```

![alt tag](https://i.imgur.com/nrD1jBf.png)

`http 拉流`

HLS (HTTP Live Streaming) 延遲比較高.

(比較不會被防火牆擋下來, 因為是以 http 為基礎)

```cmd
ffplay http://your_ip:8088/test.m3u8
```
![alt tag](https://i.imgur.com/pxgNhEB.png)

如果你到容器內的 `tmp/hls` 底下觀看,

你會發現有一個 `.m3u8`, 然後很多個 `.ts`,

`.m3u8` 是一個索引檔案, 會自動去尋找對應的 `.ts` 檔案.

![alt tag](https://i.imgur.com/La3QjRG.png)

除了用 ffplay 拉流之外, 也可以使用 VLC player 之類的工具觀看,

這邊用 RTMP 拉流示範

![alt tag](https://i.imgur.com/TVWERR4.png)

![alt tag](https://i.imgur.com/EMWbyfg.png)

其他推流

推流搭配濾鏡 (加上logo)

```cmd
ffmpeg -re -i project_view.mp4 -i demo.jpeg -filter_complex "overlay=W-w" -c:v libx264 -c:a aac -r 30 -preset ultrafast -f flv rtmp://your_ip:1935/live/test_logo
```

這邊的 stream key 設定為 test_logo

`-r` set frame rate

`-filter_complex overlay=W-w` 設定影片的濾鏡, 加在右上角

`-preset ultrafast` 代表編碼速度

拉流指令如下

```cmd
ffplay http://your_ip:8088/test_logo.m3u8
```

或

```cmd
ffplay rtmp://your_ip:1935/live/test_logo
```

![alt tag](https://i.imgur.com/w8LWmET.png)

推流 (將電腦鏡頭的影像以及聲音推流到 server)

找到 audio interface

```cmd
arecord -L
```

`-L` 代表列出所有的裝置

找到 webcam device

先安裝 v4l-utils

```cmd
sudo apt-get install v4l-utils
```

列出全部的 webcam device

```cmd
v4l2-ctl --list-devices
```

指令 ( 將電腦鏡頭的影像以及聲音推流到 server )

```cmd
ffmpeg \
    -f video4linux2 -framerate 25 -video_size 640x360 -i /dev/video0 \
    -f alsa -ac 2 -i default \
    -c:v libx264 -b:v 1600k -preset ultrafast \
    -x264opts keyint=50 -g 25 -pix_fmt yuv420p \
    -c:a aac -b:a 128k \
    -vf "drawtext=fontfile=DejaVuSansMono-Bold.ttf: \
text='CAMERA %{localtime\:%Y-%m-%dT%T}': fontcolor=white@0.8: fontsize=20: x=100: y=10: box=1: boxcolor=black: boxborderw=8" \
    -f flv "rtmp://your_ip:1935/live/test_webcam"
```

這邊的 stream key 設定為 test_webcam

`-i /dev/video0` 我的視訊裝置

`-i default` 我的音源裝置

`-b:v` 代表 audio bitrate

`-b:a` 代表 video bitrate

`-vf` 設定影片的濾鏡

拉流指令如下

```cmd
ffplay http://your_ip:8088/test_webcam.m3u8
```

或

```cmd
ffplay rtmp://your_ip:1935/live/test_webcam
```

![alt tag](https://i.imgur.com/sKb0Pfp.png)

來看一下 RTMP Server 的狀態, 總共推了3個流

`test` 正常影片

`test_logo` 影片加上logo濾鏡

`test_webcam` webcam

![alt tag](https://i.imgur.com/HYFLK35.png)

## 其他參考文章

* [alfg/docker-nginx-rtmp](https://github.com/alfg/docker-nginx-rtmp) ,

容器內有 FFmpeg 的 nginx-rtmp,

更建議去參考裡面的設定 [master/nginx.conf](https://github.com/alfg/docker-nginx-rtmp/blob/master/nginx.conf) ,

他的作法是透過 nginx server 裡的 FFmpeg 再處理後轉發出去 (當然也有可能是轉發到其他雲空間).

* [Streaming video and audio of an USB webcam](https://fschuindt.github.io/blog/2020/12/31/streaming-video-and-audio-of-an-usb-webcam-to-multiple-users-of-a-website-with-ssl-basic-authentication-and-invideo-timestamps-ffmpeg-rtmp-nginx-hls-mpeg-dash.html)

## Donation

文章都是我自己研究內化後原創，如果有幫助到您，也想鼓勵我的話，歡迎請我喝一杯咖啡:laughing:

![alt tag](https://i.imgur.com/LRct9xa.png)

[贊助者付款](https://payment.opay.tw/Broadcaster/Donate/9E47FDEF85ABE383A0F5FC6A218606F8)