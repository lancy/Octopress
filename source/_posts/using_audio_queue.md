# Using Audio Queue -- Recorder -- (Part 1)

## 什么是Audio Queue
简单的说，audio queue，并不是播放或者录音的API，它是比这更底层的东西，但他的确是播放器和录音器所使用的。当我们使用audio queue的时候，不像是使用一个录音器，而更像是制造一个录音器。
有关Audio Queue更详细的介绍，请看官方文档Audio Queue Service Programming Guide。这里主要描述实现。

## 
查看文档，我们发现这里有许多有关AudioQueue functions。
比如，创建函数: AudioQueueNewInput() 和 AudioQueueNewOutput(); 控制函数: AudioQueueStart() 和 AudioQueueStop()。参数和属性的setters和getters，等等。

先来看AudioQueueNewInput()：

    OSStatus AudioQueueNewInput (
       const AudioStreamBasicDescription  *inFormat,
       AudioQueueInputCallback            inCallbackProc,
       void                               *inUserData,
       CFRunLoopRef                       inCallbackRunLoop,
       CFStringRef                        inCallbackRunLoopMode,
       UInt32                             inFlags,
       AudioQueueRef                      *outAQ
    );

* 格式* callback function* 一个指向“user data”的指针，用来提支持callback function* callback的Core Foundation run loop* run loop mode* 必须设置成0的“Flags”* 一个指向接受新创建的AudioQueueRef的指针

我们因此可以设想一个recorder的example app需要提供下面这些内容：

* User data pointer，这是用来给你的callback function提供内容的，比如要写入的文件，在这里C惯例是顶一个struct，然而，若在一个Objective-C Class里面，你也可以传一个实例变量，或者self
* callback函数
* 你需要在主函数里，做下面这些事
    * 设置一个audio format，（AudioStreamBasicDescription）
    * 创建一个audio queue
    * Starts the queue
    * Stops the queue
    * cleanup工作，比如关闭文件
* 一些可复用和模块化的便利函数

## Sample App
