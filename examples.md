#### 视频截取

##### 单条

准确切割 + 耗时：

`ffmpeg -ss 00:01:01 -i input.mp4  -t 4 -c:v libx264 cut_output.mp4`

`ffmpeg -ss 00:01:01 -i input.mp4  -t 4 cut_output.mp4`

根据关键帧切割 + 快速：

`ffmpeg -ss 00:01:01 -i input.mp4  [-t 4 不一直切到结尾] -c:v copy cut_output.mp4`

##### ~~拼接~~

1. 裁剪 （解码+编码模式）开始点 ~ 开始点向后最近的 key frame
2. 裁剪 （复制模式）开始点向后最近的 key frame ~ 结尾点
3. 拼接 **1** 和 **2**

用ffprobe输出所有的frame，再去找到那个帧的点，然后再

有点麻烦



```
ffmpeg -i SOURCE.mp4 -t 00:02:00 -c:v copy -c:a copy a.mp4

ffmpeg -i SOURCE.mp4 -ss 00:02:00  -c:v copy -c:a copy b.mp4

(echo file 'a.mp4' & echo file 'b.mp4') | ffmpeg -protocol_whitelist file,pipe -f concat -safe 0 -i pipe: -codec copy d.mp4
拼接的时候 d.mp4的 00:02:00 处出现了画面不动，声音正常的现象
```



##### ~~插入关键帧~~

用了copy之后，就插入不了了

不用copy，就要从头decode一遍，慢的要死





> FFMPEG concepts
>
> 容器（container）：就是文件格式，在视频文件进入处理后，我们会给这个视频文件一个抽象，这个抽象就是存放这种视频文件的容器，在FFMPEG中，用来抽象文件格式的容器就是AVFormatContext；
>
> 
>
> 数据流（stream）：数据流就是我们平时看到的多媒体数据流，它包含几种基本的数据流，包括：视频流、音频流、字幕流；按照我的理解，这三种基本的数据流在时间轴上交错放置，只有这样才能满足多媒体数据流边接收边播放；数据流在FFMPEG中的抽象为AVStream。
>
> 
>
> 解复用器或者说分流器（demuxer）：FFMPEG将要处理的多媒体文件看成多媒体数据流，先把多媒体数据流放入容器（AVFormatContext），然后将数据流送入解复用器（demuxer），demuxer在FFMPEG中的抽象为AVInputFormat，我更愿意把demuxer称为分流器，因为demuxer就是把交错的各种基本数据流识别然后分开处理，将分开的数据流分别送到视频、音频、字幕编解码器处理。
>
> 
>
> 数据包（packet）当然分开的数据流在送往编解码器处理之前，要先放于缓存中，同时添加一些附属信息例如打上时间戳，以便后面处理，那么这个缓存空间就是数据包；由于数据流是在时间轴上交错放置，所以所有的视频、音频、字幕都被分割成一段一段的数据，这些一段段的数据从数据流中解析出来之后，就是存放在各自的packet，那么在这里要说明一下，单纯的视频数据包来说，一个视频数据包可以存放一个视频帧，对于单纯的音频帧来说，如果抽样率（sample-rate）是固定不变的，一个音频数据包可以存放几个音频帧，若是抽样率是可变的，则一个数据包就只能存放一个音频帧。
>
> 
>
> 在H264中没有数据包的概念，估计是因为H264只对视频做处理，所以不用分流，也就不用分流之后的缓存。
>
> 
>
> 数据帧（frame）：终于到数据帧了，编解码器真正处理的数据就是数据帧，帧这个东西，就是为了方便处理图像信息抽象出来的概念，按照我自己的理解，就是原始的图像数据信息+很多附加的信息=帧；附加的信息就是便于后面编码的信息，例如帧的解码时间、帧的显示时间等，这些信息都是为了编解码和播放的方便添加上去的，当然帧的附加信息远多于这两个信息，有兴趣的可以去看AVFrame，这是帧在FFMPEG中抽象。
>
> 
>
> 我们看到多媒体文件很大，就是因为这些附加信息也占据了很大的存储空间，此外还有协议等信息，当然没有这些附加信息，这原始的或压缩的数据流还真不好处理，例如怎么分辨上一帧与下一帧，这就要靠这些附加信息了。
>
> 原文链接：https://blog.csdn.net/beitiandijun/article/details/8280448
>
> 
>
> 音视频编解码概念
> 音视频格式有很多种，我们所熟知的音频文件有wav、mp3等 ，视频格式有mp4、3gp、rmvb、avi、mov等等。这些格式并不是只是文件的后缀不同，而是文件中的内容有很大的不同，哪怕这个媒体文件播放起来我们看起来觉得它们是一模一样的。
> 另外，我们看到的电影或者视频片段，它往往是由两个或者两个以上的流组成的，比如声音流、视频流、字幕等。甚至声音也有左声道、右声道什么的。
> 那么这些东西是怎么串联的呢？编码、解码、混流、解复用、过滤器这些都是些啥？为啥要有这些步骤？
>
> 编解码
> 我们应该都熟悉RGBA的色彩，用红绿蓝再加上透明度可以组成我们人眼可见的各种色彩，但是实际上许多色彩的差异并不是那么明显，就算肉眼去很仔细的比对都不能分辨出来。而通常视频依照24帧每秒的频率存储图像、甚至以30帧每秒的频率存储图像，如果以rgba的格式存储，占用的存储空间是非常庞大的，而且不利于网络传输。而YUV格式，如果一帧RGBA的图像数据占用16份，那么RGB占用12份，而YUV420只占用6份。虽然RGB的图像变为YUV格式质量通常会有损失，但是这个损失通常是可以接受的。
> 虽然转换成YUV格式，可以节省一些存储空间，但是情况依旧不容乐观。人们又发现对于连续的视频而言，通常连续的帧之间，图像之间存在大量重复的数据，这样，就可以在存储的时候把重复的数据去掉，再需要展示的时候再恢复出来，这样就可以大大节省存储空间，网络传输的之后，也同样把大量重复的去掉，接收端播放的实话再恢复，以达到节省带宽，加快传输速度的目的。这个去除大量重复数据的过程我们可以理解为音视频的编码，把去除的数据恢复回来的过程，我们可以理解为音视频的解码，也就是Codec。
>
> 混流、解复用
> 混流也就是复用，或许更为标准的叫法应该是复用。混流是什么？上面说了，我们看到的视频通常有图像又有声音，这些声音和图像的数据通常是两条不同的流，把这两条流混合在一起的过程就是混流。那么反之，把混合在一起的音视频拆分成各自的音频流、视频流的过程就是解复用。编解码的目的很明确，那么混流和解复用的目的又是什么呢？拿网络传输来说，在线观看视频，总不能先把声音传过去再传图像，或者说先把所有的图像传过去再传声音，而是传一点图像就传一点声音，这就是混流了，也就时Mux，与之对应的就是解复用Demux.
>
> 过滤器
> 我总是觉得这个翻译，对于Filter来说并不好听，我更喜欢翻译它为滤镜，更确切的说，应该可以把它理解为处理器。它并不是真的就是用来过滤，像把浑浊的水过滤成清水那样。比如我们给视频图像加个水印，或者改变视频的声音，让声音变得尖锐或者低沉等等，这些就是滤镜的工作了。
>
> 音视频编解码方式
> 简单的来分，我们可以把音视频的编解码分为软编解码和硬编解码，这里的软就是指软件，硬就是指硬件。所谓的软编解码，就是依赖软件，利用CPU进行编解码。而硬编解码可以理解为使用非CPU进行编解码，更准确的是，依赖音视频编解码模块进行音视频编解码。至于使用时到底是使用硬编解码还是软编解码，还是要依赖需求，根据它们各自的特点进行选择。
> 通常来说，软编解码实现直接简单，参数调整方便，支持格式广泛，编码视频质量相对硬编码更为清晰。但是软编解码相对硬编解码性能略差，会给CPU带来较重负载。
> 以Android平台为例，Android硬编解码MediaCodec从Android4.1才开始支持，混流用的MediaMuxer到Android4.3才开始支持。在Android4.1以前，如果开发者想要做视频相关应用通常会选择FFMpeg之类的开源编解码框架进行软编解码。当然，FFMpeg也可以配置选择使用硬编码。一般来说，如果应用只需要支持Android 4.3及以上，只需要做录制视频、播放Mp4文件，硬编解码方案优于软编解码方案。但是如果应用需要支持4.1以前的系统，或者要求支持其他音视频格式，比如rmvb之类的，可能就不得不使用软编方案了。
>
> 原文链接：https://blog.csdn.net/junzia/article/details/78167605
>
> ## I、P、B 帧
>
> I 帧、P 帧、B 帧的区别在于：
>
> - I 帧（Intra coded frames）：I 帧图像采用帧内编码方式，即只利用了单帧图像内的空间相关性，而没有利用时间相关性。I 帧使用帧内压缩，不使用运动补偿，由于 I 帧不依赖其它帧，所以是随机存取的入点，同时是解码的基准帧。I 帧主要用于接收机的初始化和信道的获取，以及节目的切换和插入，I 帧图像的压缩倍数相对较低。I 帧图像是周期性出现在图像序列中的，出现频率可由编码器选择。
> - P 帧（Predicted frames）：P 帧和 B 帧图像采用帧间编码方式，即同时利用了空间和时间上的相关性。P 帧图像只采用前向时间预测，可以提高压缩效率和图像质量。P 帧图像中可以包含帧内编码的部分，即 P 帧中的每一个宏块可以是前向预测，也可以是帧内编码。
> - B 帧（Bi-directional predicted frames）：B 帧图像采用双向时间预测，可以大大提高压缩倍数。值得注意的是，由于 B 帧图像采用了未来帧作为参考，因此 MPEG-2 编码码流中图像帧的传输顺序和显示顺序是不同的。
>
> 也就是说，一个 I 帧可以不依赖其他帧就解码出一幅完整的图像，而 P 帧、B 帧不行。P 帧需要依赖视频流中排在它前面的帧才能解码出图像。B 帧则需要依赖视频流中排在它前面或后面的帧才能解码出图像。
>
> 这就带来一个问题：在视频流中，先到来的 B 帧无法立即解码，需要等待它依赖的后面的 I、P 帧先解码完成，这样一来播放时间与解码时间不一致了，顺序打乱了，那这些帧该如何播放呢？这时就需要我们来了解另外两个概念：DTS 和 PTS。
>
> ## DTS、PTS 的概念
>
> DTS、PTS 的概念如下所述：
>
> - DTS（Decoding Time Stamp）：即解码时间戳，这个时间戳的意义在于告诉播放器该在什么时候解码这一帧的数据。
> - PTS（Presentation Time Stamp）：即显示时间戳，这个时间戳用来告诉播放器该在什么时候显示这一帧的数据。
>
> 需要注意的是：虽然 DTS、PTS 是用于指导播放端的行为，但它们是在编码的时候由编码器生成的。
>
> 当视频流中没有 B 帧时，通常 DTS 和 PTS 的顺序是一致的。但如果有 B 帧时，就回到了我们前面说的问题：解码顺序和播放顺序不一致了。
>
> 比如一个视频中，帧的显示顺序是：I B B P，现在我们需要在解码 B 帧时知道 P 帧中信息，因此这几帧在视频流中的顺序可能是：I P B B，这时候就体现出每帧都有 DTS 和 PTS 的作用了。DTS 告诉我们该按什么顺序解码这几帧图像，PTS 告诉我们该按什么顺序显示这几帧图像。顺序大概如下：
>
> ```
>    PTS: 1 4 2 3   DTS: 1 2 3 4Stream: I P B B
> ```
>
> ## 音视频的同步
>
> 上面说了视频帧、DTS、PTS 相关的概念。我们都知道在一个媒体流中，除了视频以外，通常还包括音频。音频的播放，也有 DTS、PTS 的概念，但是音频没有类似视频中 B 帧，不需要双向预测，所以音频帧的 DTS、PTS 顺序是一致的。
>
> 音频视频混合在一起播放，就呈现了我们常常看到的广义的视频。在音视频一起播放的时候，我们通常需要面临一个问题：怎么去同步它们，以免出现画不对声的情况。
>
> 要实现音视频同步，通常需要选择一个参考时钟，参考时钟上的时间是线性递增的，编码音视频流时依据参考时钟上的时间给每帧数据打上时间戳。在播放时，读取数据帧上的时间戳，同时参考当前参考时钟上的时间来安排播放。这里的说的时间戳就是我们前面说的 PTS。实践中，我们可以选择：同步视频到音频、同步音频到视频、同步音频和视频到外部时钟。
>
> http://www.samirchen.com/about-pts-dts/





>https://zhuanlan.zhihu.com/p/36109778
>
>https://trac.ffmpeg.org/wiki/Seeking
>
>ffmpeg 截取视频的时候，先要seeking到截取开始点，然后开始读取视频流并复制。
>
>
>
>有2种seeking （如何找到起始时间点）模式：
>
>	- output seeking
>	- input seeking
>
>对于截取视频来说，**input seeking** 使用的是 **key frame**，output seeking使用的也是**key frame**！
>
>
>
>也有2种 读取并复制 coding 模式：
>
>	- transcoding 
>	- stream copying（ffmpeg -c copy）
>
>当使用transcoding的时候，是**frame-accurate**的，而当使用stream copying的方式时，这种方式就**不是frame-accurate**的。
>
>
>
>因为是从视频到视频，并**不必须**需要 使用 transcoding（decoding + encoding），比方说我从原始的 h264 视频截取出来一小段 h264 视频，格式不变的话，就只需要使用 stream copying 模式。
>
>transcoding 就是先 decoding 再 encoding（输入是容器level，所以其实顺序是demuxing、decoding、filter、encoding、muxing），decoding 和 encoding 可以加入 filter（因为filter只能工作在未压缩的data上）；
>
>而stream copying 则是不需要decoding + encoding的模式，由命令行选项 -codec 加上参数 copy 来指定（-c:v copy ）。在这种模式下，ffmpeg在video stream上就会忽略 decoding 和 encoding步骤，从而只做demuxing和muxing。通常是用来修改容器（container） level的元数据，如下图所示：
>
>```reStructuredText
> _______              ______________            ________
>|       |            |              |          |        |
>| input |  demuxer   | encoded data |  muxer   | output |
>| file  | ---------> | packets      | -------> | file   |
>|_______|            |______________|          |________|
>```
>
>因为没有transcoding的过程，所以速度非常快（相比于有transcoding的过程）。
>
>
>
>为什么stream copy不是frame-accurate的？ 
>
>**对于ffmpeg来说，当使用-ss 和-c:v copy 时， ffmpeg将只会使用i-frames**。比方说（当output seeking + stream copying的时候）你指定起始点是719秒，而直到721秒才有个key frame，那么cut产生的视频在前2秒就只有声音，过了2秒后才会有视频（刚好到了key frame），所以一定要小心啊（注意啊，mplayer软件可能会把2秒空白期挪到后面）。
>
>以截取一段4秒长的视频为例（选取00:01:01，也就是起始点为61秒，是因为此处最近的关键帧位于58.56秒和64.56秒）：
>
>```text
>#1, use stream copying & input seeking
>ffmpeg -ss 00:01:01 -i gemfield.mp4 -t 4 -c copy cut1.mp4
>
>#2 use stream copying & output seeking
>ffmpeg -i gemfield.mp4 -ss 00:01:01  -t 4 -c:v copy cut2.mp4
>
>#3 use transcoding & input seeking
>ffmpeg -ss 00:01:01 -i gemfield.mp4  -t 4 -c:v libx264 cut3.mp4
>
>#4 use transcoding & output seeking
>ffmpeg -i gemfield.mp4 -ss 00:01:01 -t 4 -c:v libx264 cut4.mp4
>```
>
>这#1、#2、#3、#4分别表现是什么呢？
>
>\#1，当为Input seeking + stream copying的时候，我们想截取的是[61, 65)的片段，实际截取的是**[58.56, 65)**的片段，是的，ffmpeg往前移动到了一个I-Frame；
>
>\#2，当为output seeking + stream copying的时候，我们想截取的是[61, 65)的片段，实际截取的是**[64.56, 65)** 的画面，**再加上 (4 - 0.44)秒的空白片段**，生成长度为4秒的视频。播放器在播放的时候，这个空白片段怎么播放是由播放器自定义的。总之，画面的有效信息只有后面的关键帧开始的一小段信息；
>
>\#3，当为input seeking + transcoding 的时候，我们想截取的是[61, 65)的片段，实际截取的是**[61, 65)**的片段，是的，frame-accurate；
>
>\#4，当为output seeking + transcoding 的时候，我们想截取的是[61, 65)的片段，实际截取的是**[61, 65)**的片段，是的，frame-accurate。
>
>可以看到，#3和#4是一样的。



##  剪切画面:

```
ffmpeg -i .\out.mp4 -vf crop=2048:1024:0:0 singleRight.mp4

crop=w:h:start_x:start_y
w: 视频的宽 h: 视频的高  (start_x,start_y): 矩形开始的点


裁剪的时候如果用了gpu的解码器后, 输出的视频没有被裁剪, 很奇怪, 参数的位置写错了?:
ffmpeg.exe -hwaccel cuvid -c:v h264_cuvid  -i .\test.mp4 -vf crop=2048:2048:0:0 -c:v h264_nvenc -y .\singleLeftTest.mp4
去掉-hwaccel就好了:
ffmpeg.exe -c:v h264_cuvid  -i .\test.mp4 -vf crop=2048:2048:0:0 -c:v h264_nvenc -y .\singleLeftTest.mp4


如果不在-i前面指定 -c:v 则会使用cpu去解码视频, 但是后面的-c:v是生效的, 于是就会用gpu编码;
而且speed会变到原来的2倍, 目测任务管理器, CPU满载, GPU编码器一半, GPU编码器无.
ffmpeg.exe -hwaccel cuvid -i .\out.mp4 -vf crop=2048:2048:0:0 -c:v h264_nvenc -y .\singleLeft.mp4
```

#### 苏肤佳 录播裁剪 剩下歌词滚动和歌曲进度

`ffmpeg.exe -c:v h264_cuvid -i .\20年2月录播_P1_18日：被套了虚弱的一天.flv -vf crop=1000:920:90:115 -c:v h264_nvenc -b:v 0.5M -c:a copy -y small.mp4`



## 使用显卡

```
ffmpeg.exe -hwaccel cuvid -resize 960x540 -c:v h264_cuvid -i .\origin.mp4 -b:v 0.8M -c:a copy -c:v hevc_nvenc out6_0.8M_265.mp4

-hwaccel 使用显卡加速
cuvid 使用的是 G卡
-resize 改变视频比例
-c:v h264_cuvid 视频解码的时候 用G卡的解码器
-b:v 0.8M 把视频的比特率保持在0.8M左右
-c:v hevc_nvenc 使用h265的 G卡的编码器, 也可以用 h264_nvenc也就是h264标准

```



# 批量操作

```powershell
$dir = dir .
foreach($_ in $dir) {
  ffmpeg.exe -c:v h264_cuvid -i $_.name -s 960x540 -c:v h264_nvenc -c:a copy -b:v 1M   ($_.name+'small.mp4')
}
```

