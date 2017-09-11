![](http://upload-images.jianshu.io/upload_images/1110736-7864e42a7f26ed3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)



**为什么要逆向rom层？**
 先解释下我们为什么要这样做吧，`Android`的碎片化不用我说大家都懂的，五花八门。时常应用层开发者痛不欲生，明明在我这里开发时运行没有问题，tm的一上线就出问题。为什么？这个机型`rom`被改动了呀。真好烦哦。这种说bug又不是bug，但确实又是bug的问题还得你修。特别是某些小公司条件限制没能力买齐全主流的机型供开发人员工作使用。（**想要猪儿肥又壮，光吃草，还不给喂饱！！！**）而你还得默默的给适配好，怎么办？办法也不是没有，业界内我在网上资料查找少又公开这个思路，不过我觉得如果能逆向一把厂商的`rom`，（估计大厂的爱paipai适配应该也会这样做，因为我没有在大厂待过呀。所以只能估计了。）看看`rom`厂商的实现怎么写的，难道还不能适配好了嘛！所以就有了下文。

这篇文章拖了很久，大纲我是一个月前写好的，一直没来得及填充内容。罪过，醉过！~ 工作上实在抽不出时间了，一回家倒床就迷迷糊糊睡着了。后面有同学问我说:"你咋还没发布？那个逆向适配的，我等着打赏一波呢。”听完这句话，我又忍不住我的麒麟臂。乘着周末，来，快扶老夫起来，开整。

为了能保证后面看文章的同学能够顺利跟着本文去实践，我清空了电脑上已有所有反编译工具环境，从头来过一遍，（其实就是从头再踩一遍坑啦。经验告诉我们，做这种事情，从来就不会那么顺利。(┬＿┬)哭~）已尽量保证本章在写作的过程中不会出现疏漏而导致的大家掉坑里了，但是不可避免还是可能会疏忽笔误之类的，那么大家可以给我留言，可能回复慢一点，但是一定会认真回答的。

 感谢白手、追风917、imesong 提供的帮助，谢谢 。还有林学森大大的一本书推荐《深入理解 Android 内核设计思想》，对本篇文章提供了学习参考。

**温馨提示：前方长篇预警，看不完，又觉得精彩的时候。收藏一下点个喜欢吧。咱们慢慢来哈。**

-----

# 目录
- 基础知识
  - dex
  - odex
  - smali

- rom层应用分析
  - odex与apk
  - 框架文件

- 逆向工具箱
  - baksmali
  - dex2jar
  - apktools
  - gd-gui
  - SVADeodexerForArt
  - apkDB

  - 当前Activity


- 实战逆向MIUI系统设置WIFI
  - 需求
  - 思路
  - 寻找切入点
  - 下载固件
  - 拆包与合并资源
  - 翻源码
  - 反射API


----

## 基础知识

基础这部分特别重要，如果不是很扎实同学一定要认真学习噢。熟悉的同学就往后跳着看吧，为了照顾大多数同学，这部分就不能丢了。

- dex

`dex`大家应该不会陌生，我们平时写的类都会被转换成`class`然后打包成`dex`。你可以尝试解压缩你的`apk`文件查看。

也可以下面的方式转换`java`类直接转换成`dex`，步骤如下：

>1.`javac xxx.java  ` （得到>>> `xxx.class`文件）
2.配置环境变量 Android SDK开发包下的dx工具 （路径:`/build-tools/编译版本/dx` ）
3.`dx --dex --output=xxx.dex xxx.class`  （得到>>> `xxx.dex`文件）

这样得到的dex是可以直接用Android虚拟机执行的，可以使用`adb shell`进入并输入` # dalvikvm -cp /sdcard/name.dex name` 可直接运行。

- odex

`odex???`,没毛病。用百度百科原话说：

>`ODEX`是安卓上的应用程序`apk`中提取出来的可运行文件，即将APK中的`classes.dex`文件通过`dex`优化过程将其优化生成一个`dex`文件单独存放。

如果经常刷机的朋友肯定不会陌生，经常有民间`rom`作者说已优化`odex`。那么其实他就是把`apk`包中的`dex`提取出来转成`odex`。这个过程为什么说是优化呢？因为在 `4.4ART`虚拟机之前，`DVM`虚拟机每次打开一个应用程序前都需要先解压缩包，然后杀鸡取卵的拿到`dex`再读到虚拟机内。这个过程中解压是要花时间的。为了让开机和启动应用速度加快。将`apk`中的`dex`提前取出来就省去了解压的耗时。第二一个以前很老的机器往往可供系统用的存储空间不够大，常规的做法除了提前拿到`odex`以外，还会把`apk`包中`dex`删除掉，已节约空间。不得不说`rom`工程师都是在绞尽脑汁的做优化。也许就是只为提升那0.1秒的时间吧，也要不折手段做到极致。

- smali

smali是虚拟机指令语言，是给虚拟机看的。我们如果把dex文件反编译后就可以看到。里面的指令集，读者无需深究这部分的指令集。本篇也不涉及这些内容。我们了解个大概就行了。这些工作就留给专门的反编译工作者吧。（你要深入学，也没人会拦着你的。哈哈）

好，这里我们了解了三个基础的东西，我们来总结一下。
**`dex、 odex、smali`**
我们脑子里过一下，有个映像知道它们都是干嘛的就可以了。下面我们就进入`rom`层的逆向流程，干货来了，准备好哦。

-----

## rom层应用分析

**提示：**本章只是在粗糙的分析系统应用结构，不涉及从`rom`包下载到解压和找应用路径的来龙去脉，在后面的实战章节中会细细讲解，读者无需纠结下面要讲到的app从哪里变出来的。（问题是我也变不出来呀。QAQ）

- odex与apk
为什么`apk`要和`odex`合并？前面我们在上以一个基础篇章节讲过，`odex`的优化。`rom`厂商的会把`app`把系统级别的`App`的源码和资源文件做分离。我们来看一幅图。

![](http://upload-images.jianshu.io/upload_images/1110736-8e0d455022210de1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面这幅图左边是常规APP，右边是小米SettingsAPP。可以清楚的看到Settings里面没有`classes.dex`文件。如果按照往常应用层打包发布apk的思路走。apk内至少会有一个`classes.dex`。（dex分包会有多个）
而我们知道一个app里的源码就在dex里，如果用常规的应用层反编译去反编译则什么都看不到，还会导致一些反编译工具抛出无法找到dex的异常。这个问题先放到这里。

我们再看另一附图：

![](http://upload-images.jianshu.io/upload_images/1110736-3b25c7242b2a8b7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

老司机可能一眼就看到了这个目录非同寻常，没错，正是MIUI的`rom`解压缩后的目录。其中`\system\priv-app\Settings`是系统设置应用的路径。我们发现其中有一个`oat\arm`目录打开后可以发现里面有一个Settings.odex文件。而这个文件正是`Settings.app`的源码文件。

- 框架文件
往常我们的apk中会包含一个`Resources.arsc`文件，简单来说它是App的资源文件索引表，包括xml、图片、文件等索引都被存在内。而系统应用除了在`Resources.arsc`有索引，还会跟framework的资源产生关联。由于我不是做`framework`层开发的，可能存误区，欢迎各位指正。

框架文件是指`framework的framework.apk、core.jar`文件等。在反编译系统应用时必须要借助他们做资源映射。


![](http://upload-images.jianshu.io/upload_images/1110736-42b3a996cc17db91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/1110736-4a7c9babe27c42fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rom层对系统应用进行了`odex`优化，其中就包括了资源文件的依赖，有一部分是存放在`framework.apk、core.jar`等。并不像我们平时开发app的时候只存在于`Resources.arsc`文件中。不过因为厂商和系统版本不同，它们的名称可能也有点差别，没有关系。总之我们后面不管是合并odex和apk的时候，或者反编译源码的时候就需要使用到它们。大家留个映象看后面的操作。对应的目录是：`\system\framework`

![](http://upload-images.jianshu.io/upload_images/1110736-b714d2482d1be616.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们可以看到里面并没有包含`dex`文件，只有`res`和`assets`资源文件。

-----

## 逆向工具箱
下面我们一起来看一下现在主流的反编译工具，我会依依简单介绍。

- baksmali
`baksmali`用于将`odex`解析成`smali`的工具。可能改个名字叫（`odex2dex`更合适）

- dex2jar
 将dex转换成jar文件。

- gd-gui

反编译整个`jar`可以对把smali文件解析成`java`源码查看的一款工具。

- apktools
拥有对`apk`编译、反编译、签名等功能。

- SVADeodexerForArt
自动合并框架工具，可以通过简单的配置，就可以合并出`rom`里的`app`和`odex`文件。

- apkDB
中文名称安卓逆手，整合了反编译多款工具的一个工具集合，功能很强大，几乎你想要的一切操作都能在里面找到对应的工具命令。

- 当前Activity
这是一款app应用，用于显示当前手机界面上的`Activity`等包名信息。


**工具下载地址：**
**[baksmali](https://github.com/JesusFreke/smali)**
 **[dex2jar](https://github.com/pxb1988/dex2jar)**
**[JD-GUI](http://www.softpedia.com/get/Programming/Debuggers-Decompilers-Dissasemblers/JD-GUI.shtml)**
**[Apktool](https://ibotpeaches.github.io/Apktool/)**
**[SVADeodexerForArt](http://htcui.com/download.php?id=7791)**
**[apkDB](https://www.idoog.cn/?p=2933)**
**[当前Activity](https://play.google.com/store/apps/details?id=com.willme.topactivity&hl=zh_CN)** (需要过墙)

-----

## 实战逆向MIUI系统设置WIFI

前面的知识比较揉杂，下面我们通过一个实际案列来分析具体rom逆向过程。

- 需求

 我们有一个这样的需求，需要在应用内切换WIFI。
这个需求很普通很正常吧？没错，我也是这样想的。但是呢，不久你就会发现部分小米高版本机型切换wifi的API调用了不起作用。我们来看一眼代码。

https://developer.android.com/reference/android/net/wifi/WifiManager.html
![](http://upload-images.jianshu.io/upload_images/1110736-fc3e805ac42db49e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是官方提供的切换热点的API。

![](http://upload-images.jianshu.io/upload_images/1110736-1d41f28648ab2023.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们拿起来直接调用了。这一切都看似正常。可就是在小米上出现这种问题。遇到这个问题，我们先静下心来分析下。

除了小米还有别的机型也有问题？
跟android版本也有关系，API高版本变更？


第一个问题可以暂时排除放一边，能测的都测了都没毛病，上云测平台也试过了。


第二个问题关于变更情况。下面是官方文档，6.0WLAN的变更情况。

![](http://upload-images.jianshu.io/upload_images/1110736-51050b0dea2c04c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注意：**我翻译下，官方的意思是说由用户手动创建和其他app创建的`WifiConfiguration`配置信息是不能被我们修改的。明白了吗？和切换没有什么关系。因为我们不修改，只是连接切换操作。

- 思路

这下就比较操蛋了，既然`api`没有做切换限制为何不能切`wifi`。带着这个疑问我想了想别家的应用是如何做的呢？比如WIFI万能钥匙。我如果反编译他们家的APP会被加壳和代码混淆的。（其实我尝试过了，但没有脱开。）

想问题掉了一地头发后，最后我想到了为什么系统的设置他自己又能切？是不是有隐藏的`API`调用。就这样半猜测，半怀疑的态度去打算搞事情了。首先得去拿到系统设置APP的源码吧？我得反编译回来`rom`里的设置APP。

说一个小插曲：文章里看起来还算顺畅，但是实际操作的时候各种艰难险阻。问题重重。在APK导出系统设置程序这里浪费了不少时间，一直天真的认为系统的apk也能直接导出来反编译的。结果是压根不知道`odex`优化了。（什么你也不知道`odex`优化？说明你没有看我前面章节讲到的东西。）

- 寻找切入点

这里我们一步步来，首先打开系统设置，跳转到WIFI设置界面。
它是酱样子的。

![](http://upload-images.jianshu.io/upload_images/1110736-1ab716a957afeac6.png?imageMogr2/auto-orient/strip%7CimageView2/2/h/400)

接下来我们需要知道这个`Activity`的包名和`fragment`的叫什么。
打开Android ADB shell
输入：**`dumpsys activity top`**
如下图所示，我们可以清晰的看到我们需要的信息。
**Activity:** com.android.settings.SubSettings
**Fragment:** MiuiWifiSettings
![](http://upload-images.jianshu.io/upload_images/1110736-5932dd50ae4d8312.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一会儿我们反编译后的`App`就可以通过包名+名称来找到对应的切换wifi代码来。
这里除了使用`adb shell `的方式还可以用 当前`Activity App` 进行查看，只不过信息没有这么全面。

- 下载固件

我们找到MIUI官网，下载稳定版的固件就行了。

http://www.miui.com/download.html

这里我用的是米max

![](http://upload-images.jianshu.io/upload_images/1110736-6b3e7515c8833916.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下载后，我们直接解压即可，它是酱样的。

![](http://upload-images.jianshu.io/upload_images/1110736-1ee1a86a9e36cad3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 拆包与合并资源

我们先来找一下SettingsApp的路径。
这里我已经找好了，路径是：`\system\priv-app\Settings\`

如果你想问我，怎么知道的路径的？
网上搜呀，自己翻目录哇，翻烂了就出来啦。
噢，对了，在windows下有一款叫Everything的神器也可以试试，超好用的。


  我们打开SVADeodexerForArt自动合并框架工具。
选择我们解压好的目录至system下。依次全部勾选下面三个选项，点击Execute后就会出现下面这幅图的样子，只需要静静等待几分钟就可以了。他会将odex资源与apk进行合并。
![](http://upload-images.jianshu.io/upload_images/1110736-25aea0aefc55bae8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意合并odex不止这一种方式，这是目前为止最简单，最傻瓜的操作方式。
读者如果有需要深究可以尝试`baksmali.jar+ smali.jar +apktools`进行或者apkDB回编。工具下载地址在上个章节有给出。

使用列子如下：
```
java -jar baksmali.jar -d ./system/framework -x Settings.odex
java -jar smali.jar classout/ -o classes.dex
```
因为很麻烦，有很多坑，我踩了很多次。主要两个问题：
1.框架文件不齐全找不到
2.smali回编抛出奇奇怪怪的异常，好像和代码混淆还有关系，因为有一些暗桩代码
目前我也不知道怎么解决，网上也没有找到可参考的资料，所以我推荐大家直接用SVADeodexerForArt就好了，别费那个事浪费时间。


好言归于好，我们来看一下执行完后生成的目录：

会在当前工具下的根目录上out出以下几个文件夹，这里分别包含系统framework、UI、预装APP和系统APP。
![](http://upload-images.jianshu.io/upload_images/1110736-b7c32b6b27f207fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我们重点关注`SettingsApp`，打开`priv-app\Settings`,如下图所示。上面的是合并后的13905kb，下面是合并之前的12370kb。这里就可以看出已经合并成功了。
![](http://upload-images.jianshu.io/upload_images/1110736-0fae04edc0b22c00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们用压缩工具看一眼：


![](http://upload-images.jianshu.io/upload_images/1110736-abb884d791b47a31.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很明显已经合并成功了噢。
下面我们使用`ApkTools`反编译过来查看。

使用 `ApkTools `输入命令：`apktool.bat d -f Settings.apk` （注意Settings.apk的路径，我这里是拷贝到了apktool 同一个目录下）

![](http://upload-images.jianshu.io/upload_images/1110736-e57dd40a8628b692.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应会生成一个同名的文件夹叫Settings，我们来看一眼里面有什么

![](http://upload-images.jianshu.io/upload_images/1110736-dfa99937f3f0d330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简直完美，这里我们可以看到`smali`文件夹出现了。我们现在要做的就将`smali`文件合并成`dex`。

继续下一步
使用smali.jar工具
命令：`  java -jar smali-2.1.3.jar ./Settings/smali/ -o setting.dex`

![440765BBCB9145882A58EF3A2D4CD58F.png](http://upload-images.jianshu.io/upload_images/1110736-12563548f2dc5c67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将生成dex文件：

![](http://upload-images.jianshu.io/upload_images/1110736-f32f4e63e8af08ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

别急就快看到源码了。马上，我保证。

使用**[dex2jar](https://github.com/pxb1988/dex2jar)**工具，将dex转jar。

输入：`d2j-dex2jar.bat Settings.dex`
得到 `settings-dex2jar.jar`

![](http://upload-images.jianshu.io/upload_images/1110736-4d1a1837b99e2e9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们打开jd-jui工具，将jar拽入加载打开。

![](http://upload-images.jianshu.io/upload_images/1110736-0f795a0cb5e205da.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，我们就拿到了源码。好，我们稍微整理一下。一会儿一起来看源码。

1.下载rom包
2.-->解压
3.--> `SVADeodexerForArt`合并`odex`
4.-->` apktools`反编译
5.-->`smali.jar`打包`smali`成`dex`
6.-->`dex2jar.jar`转换` dex`成`jar`
7.-->`jd-jui`工具加载jar查看

至此一共七个步骤，其实这个操作并不难，只是需要借助多款工具来完成就略显步骤复杂。然后中间可能会有些小坑，卡住的话就容易让人失去信心。新手别去强记这些，用多了就自然就熟悉了。

- 翻源码
好，还记得我吗的需求吗？需要在应用内切换`wifi`。但是小米内无效。
嗯，现在我们有源码了，来，一起来看看系统怎么做的吧。
这是我们之前寻找切入点拿到的信息，我们直接通过jd-jui切入源码。

**Activity:** com.android.settings.SubSettings
**Fragment:** MiuiWifiSettings

![](http://upload-images.jianshu.io/upload_images/1110736-81f89ba9c3ae8417.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我们发现里面什么都没有，唯一有用的是它继承了`iy`，没猜错的话iy一定是一个`Activity`，这里应该是被混淆了，我们点进去看。

![](http://upload-images.jianshu.io/upload_images/1110736-f6754c45d68de4d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以发现里面比之前那层壳里的内容丰富多了，还记得我们之前分析过那里不是`Activity`直接展示的而用了`fragment`我们搜索一下`MiuiWifiSettings`。果然存在这里。

继续点过去。

![](http://upload-images.jianshu.io/upload_images/1110736-f53d4aeabbc55329.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发现又是一个空壳子，我们直捣黄龙深入一点看父类。

![](http://upload-images.jianshu.io/upload_images/1110736-87d037a199a4710c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

果不其然，持有了`WifiManager `对象，这就是用来管理wifi的对象。

![](http://upload-images.jianshu.io/upload_images/1110736-9760802cb374bbb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看看它是如何处理`wifi`切换的，最终发现`jn.connect`方法。看到没果然有自己一套。死活切不过去，原来是有隐藏`api`调用的。而`sdk`层并没有暴露出来，谷歌官方开发文档中也不存在这个方法。

![](http://upload-images.jianshu.io/upload_images/1110736-4cb41685d15cfd39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

好气哦，居然偷摸调用一下～～


- 反射API

好，既然知道了是使用了隐藏api我们可以用反射来解决，代码如下。

``` java
Class cls = mWifiManager.getClass();
  Method method = cls.getDeclaredMethod("connect", WifiConfiguration.class, Class.forName("android.net.wifi.WifiManager$ActionListener"));
        method.invoke(mWifiManager, mWifiConfiguration, null);
```

这样我们的疑问就解决了，测试在miui上通过。

到这里我们的实战就完成了。以后麻麻再也不用担心厂商改了。顶多我们麻烦一丢丢啦。不至于卡住完全被动的情况。


## 后记

对不起，我骗了大家。我是一个大骗纸。其实这个`api`不是小米私有的，是系统隐藏API。只是说常规的切换方式在小米上无效。正好小米的  `Settings`内部用的是`connect`方法系统隐藏`Api`。我们来看一眼  `WifiManager `官方源码。
https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/wifi/java/android/net/wifi/WifiManager.java

![](http://upload-images.jianshu.io/upload_images/1110736-e1028421f5db15c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我真的好害怕小米的工程师看到了这篇文章以后，表示这个锅，我们不背。所以这里要澄清一下。

然后大家的重点不要纠结这个`API`啦，思路，思路，思路很重要。关键是逆向`rom`层带来的便利真的不是一点半点的。几乎可以解决绝大部分因为厂商改动导致代码碎片化带来的疑难杂症。希望本文能给广大的应用层开发者提供一丝帮助，诶呦，我就超开心的。🙏


---
# 如何下次找到我?
- 关注我的简书
- 本篇同步Github仓库:[https://github.com/BolexLiu/MyNote](https://github.com/BolexLiu/MyNote)  (可以关注)
![](http://upload-images.jianshu.io/upload_images/1110736-f0a700624e0723ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**原创文章，未授权禁止转载。**