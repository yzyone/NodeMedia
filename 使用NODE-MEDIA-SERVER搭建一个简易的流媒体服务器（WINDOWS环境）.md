# 使用NODE-MEDIA-SERVER搭建一个简易的流媒体服务器（WINDOWS环境）

记录一下使用node-media-server的一些过程。本文章环境为windows。本文章适合初学者。

使用到的东西：nodeJs、ffmpeg、node-media-server。

这里说一点（如果有错欢迎指出）：

node-media-server是作为流媒体服务器，你可以把他理解为中转站，用于转换流格式或者对视频流做一些操作以及向外推送流地址。

ffmpeg是作为推流工具，将你需要转换的视频或者视频流推流到流媒体服务器中。

拉流的意思是从流媒体服务器上拉去视频流，观看者通过拉取流媒体服务器发布的流地址进行观看。你用视频播放器播放就是在拉流。

# 安装

node-media-server是基于node.Js开发的，所以需要先使用npm安装。

```
npm install node-media-server
```

# 编写并运行NODE-MEDIA-SERVER

新建app.js。并编写下述代码，然后运行app.js

```
const NodeMediaServer= require('node-media-server');
const config = {
    rtmp: {
        port: 1935,
        chunk_size: 60000,
        gop_cache: true,
        ping: 60,
        ping_timeout: 30
    },
    http: {
        port: 8000,
        allow_origin: '*',
    }
};

var nms = new NodeMediaServer(config)
nms.run();
 
```

可以访问localhost:8000/admin地址，访问node-media-server的管理端界面。截图如下：

![img](https://www.freesion.com/images/192/205e053f0998e21d352add0333329738.png)

# 使用FFMPEG命令推送地址。

## 推送视频文件

在执行下面的代码的时候，需要将node-media-server启动起来。

```
ffmpeg -re -i ./video.mp4 -c copy -f flv rtmp://localhost:1935/live/STREAM_NAME
```

上述的命令经过node-media-server后会产生两种流地址。一种rtmp。一种flv。前者可以在电脑上播放，后者可以在手机和电脑上播放。rtmp地址为FFmpeg里的命令地址

flv地址为： 

```
http://localhost:8000/live/STREAM_NAME.flv
```

## 推送RTSP流

推送rtsp（摄像头视频流）只要将上述的./video.mp4该一下就行。博主在测试过程中发现，推送rtsp流要么会出现绿屏要么会出现丢包现象，特别是和hls结合在一起，丢包率更大，所以不建议使用命令去推送rtsp流。

## 转HLS流格式

转hls流需要注意一点，需要指明一下mediaroot参数，虽然node-media-server内部有设置默认值，但是还是推荐在设置一次。然后使用下述配置即可。

```
const NodeMediaServer= require('node-media-server');
const ff = require('ffmpeg');
const config = {
    rtmp: {
        port: 1935,
        chunk_size: 60000,
        gop_cache: true,
        ping: 60,
        ping_timeout: 30
    },
    http: {
        port: 8979,
        mediaroot: './media/', // 建议写
        allow_origin: '*',
    },
    trans: { // 这里参数是trans参数，不是relay参数，relay参数中配置hls无效
        ffmpeg: './bin/ffmpeg.exe',//指明FFmpeg位置
        tasks: [
            {
                app: 'live',
                ac: 'acc',
                vc: 'libx264',
                hls: true,
                hlsFlags: '[hls_time=2:hls_list_size=3:hls_flags=delete_segments]',
                dash: true,
                dashFlags: '[f=dash:window_size=3:extra_window_size=5]'
            }
        ]
    }
};

var nms = new NodeMediaServer(config)
nms.run();
```

启动上述代码后，使用FFmpeg进行推流，稍等一会，你就会发现在mediaroot指向的目录下生成一个live/STREAM_NAME的文件夹，里面存放着m3u8文件。由于需要先生成m3u8文件，所以如果是推流摄像头的话，会存在比较大的延迟。

m3u8地址为：

```
http://localhost:8000/live/STREAM_NAME/index.m3u8
```

如果发现m3u8播放有问题，把ac和vc两个参数去掉试试。楼主在实际使用的时候，这两个参数并没有使用。

# 使用代码对RTSP流转流

对于有的使用者有可能需要将rtsp摄像头视频流进行推流，以便进行跨端预览，博主这里建议使用这种方法。这种方法无需使用cmd执行FFmpeg命令，而且延迟经博主测试为3s（内网，由于没有外网地址，所以外网不是很清楚）。延迟较小。

代码如下：

```
const NodeMediaServer= require('node-media-server');
const config = {
    rtmp: {
        port: 1935,
        chunk_size: 60000,
        gop_cache: true,
        ping: 60,
        ping_timeout: 30
    },
    http: {
        port: 8979,
        mediaroot: './media/',
        allow_origin: '*',
    },
   relay: {
        ffmpeg: './bin/ffmpeg.exe',
        tasks: [
            {
                app: 'live',
                mode: 'static',
                edge: 'rtsp://admin:****@192.168.4.167:554/Streaming/Channels/101',//rtsp
                name: 'technology',
                rtsp_transport : 'tcp', //['udp', 'tcp', 'udp_multicast', 'http']
            }
        ]
    },
};

var nms = new NodeMediaServer(config)
nms.run();
```

这种方法可以产生两种视频流，一种rtmp一种flv。

 

# 总结

因为博主搭建流媒体服务器主要是为了项目中对摄像头进行转流，但是之前JAVA同事有处理过发现会消耗大量的硬件资源，不过博主使用node-media-server倒是没有发现消耗多大的资源，最终还是需要各位具体测试。而且对于摄像头转流，为了避免不必要的性能消耗，楼主打算仅当观察者发起预览的时候，才让服务端启动流媒体转流功能（使用代码对rtsp流转流），当观察者关闭预览，就立即把流媒体功能关闭。这样能避免性能的不必要消耗。毕竟可以直接通过代码直接操作，这样就比较方便，也能降低服务器的一定压力。

版权声明：本文为QiZi_Zpl原创文章，遵循[ CC 4.0 BY-SA ](http://creativecommons.org/licenses/by-sa/4.0/)版权协议，转载请附上原文出处链接和本声明。

本文链接：https://blog.csdn.net/QiZi_Zpl/article/details/107259935