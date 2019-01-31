---
layout:     post
title:      使用Windbg&OllyDbg从头调试windows服务
subtitle:   Windows平台使用Windbg&OllyDbg从头调试服务
date:       2019-01-20
author:     Weizhou
header-img: img/p2_bg.jpg
catalog: true
tags:
    - Malware
    - Reverse
---

## 前言
在分析恶意软件时，恶意软件通常会创建服务，然后拉起服务删除自身文件，结束进程。很多关键的操作都在恶意软件新创建的服务之中，当然可以使用IDA进行静态的调试，但同时需要OD对服务进行动态的调试。

## 正文
对于绝大多数的服务可以直接attach到OD或者`windbg`上进行调试，但是如果想从启动就开始就调试该服务，这种attach的方法就不是很适用。网上有一些多年前的文档教程，但是在亲身尝试的时候总是用各种各样的问题，我就结合网上教程和亲身测试对如何使用`OD/Windbg`调试windows服务进行一个较为详细的说明。
### 1. 恶意软件分析：
本人分析的是一个挖门罗币恶意软件的loader，直接步入对服务进行操作与的正题，首先是创建一个服务：

[![image.png](https://i.postimg.cc/Cxpp0GXY/image.png)](https://postimg.cc/Tp04qDTH)

然后会对服务注册表中的一些值进行设定，这里就省略了。

在注册表的数值被设置之后，会直接调用`StartService()`开启服务：

[![image.png](https://i.postimg.cc/PxxBy2pb/image.png)](https://postimg.cc/CZ3PLC4d)

此时要在次设置好断点，不要执行该布步骤。

静态分析该文件时，在文件中可以看到很多的函数操作，包括对文件的读写，对进程的遍历等等。但是在单纯执行该文件的时候，这些函数操作都不会表现出来。IDA反汇编代码:

[![image.png](https://i.postimg.cc/c1VLK82k/image.png)](https://postimg.cc/z3Fr63JT)

从以上代码可知，如果该程序是以服务的形式被拉起的，则会运行`sub_401F30()`函数的程序，否则会向下运行到绿色框中的程序。绿色框中程序的功能就是，删除现有服务，再新创文件，自删除文件，创建新服务，设置注册表参数，拉起服务，结束程序。所以如果不进入到服务进行debug，则一直不能看到核心的恶意代码。而且这些恶意操作执行的速度会很快，不等你attach到OD上就会结束，所以此时需要设置当服务一被拉起就进行调试功能。

恶意软件所注册的服务通常是自启动的，为了方便调试，可以将其更改为手动开启模式。

### 2. 设置运行程序打开OD/Windbg功能
使用注册表编辑器，打开注册表路径：<br> `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Image F
ile Execution Options` ，添加一个主键，名称设置为将要调试服务的exe文件：

[![image.png](https://i.postimg.cc/0N2kgGKj/image.png)](https://postimg.cc/QFwG1TbZ)

要调试的服务的exe文件是scvhost.exe所以在
`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Image File Execution Options` 键值下新建一个键值的名称为`scvhost.exe`，然后再里面添加一个新建一个字符串选项，命名为`debugger`，最后对`debugger`这个参数进行设置，设置为调试器的路径。在设置好之后，运行`scvhost.exe`文件，就自动进入OD的调试界面。所以最好在调试前在被调试服务文件入口设置好断点。

### 3. 延长服务开启超时时间设置：
当调试服务.exe文件时，其实是在从服务启动时进行调试的，系统对启动服务设置了默认的超时时间，超过时间就会退出启动并报错：

[![image.png](https://i.postimg.cc/cLTWcKkS/image.png)](https://postimg.cc/TKyBdPX7)

系统默认启动服务的超时时间是30秒，但30秒远远不够对服务进行分析，所以要延长服务超时的时长。打开注册表，`在HKEY_LOCAL_MACHINE\SYSTEM \CurrentControlSet\Control` 键值下查找`ServicesPipeTimeout`参数，一般不存在，需要新建。新建DWORD值`“ServicesPipeTimeout”`，其值为欲设置的超时时间，先选择为十进制，数值单位是毫秒，如设置 24小时，则值为`86400000`毫秒：

[![image.png](https://i.postimg.cc/3JcC09Y5/image.png)](https://postimg.cc/YhQmJ6vb)

在设置好貌似不能直接起到作用，**需要重启** 才能有效果。

### 4. 设置要调试的服务与桌面交互：

之有打开了桌面交互功能才能够在服务被加载的时候弹出`OD/Windbg`。首先是打开服务的桌面交互功能。在`cmd`中输入，`services.msc`，打开服务控制版面，找到要被调试的服务，然后勾选桌面交互选项：

[![image.png](https://i.postimg.cc/qMz2rnnm/image.png)](https://postimg.cc/LnFJBJxf)

实际上这等同于修改注册表中的要调试服务的type选项。选择`“Type”`，修改其值为：`原值 OR 0x00000100（如原值为：0x00000010 OR 0x00000100 ＝0x00000110）`：

[![image.png](https://i.postimg.cc/vHkfZBtG/image.png)](https://postimg.cc/D8Q8C7NN)

除了要打开服务的桌面交互选项之外，还要开启服务的桌面交互检测服务，系统默认通常是关闭的，需要手动打开。如果不打开该服务，OD/windbg调试窗口就不会弹出并启动该服务：

[![image.png](https://i.postimg.cc/qvKnhT1y/image.png)](https://postimg.cc/cvs62VyL)

### 5. 启动服务

完成上述步骤都设置好之后，启动要调试的服务，会弹出如下对话框：

[![image.png](https://i.postimg.cc/JnrB2BM5/image.png)](https://postimg.cc/rdHmdsdD)

点击查看信息，此时调试器会跳出，加载到服务起始处：

[![image.png](https://i.postimg.cc/V6KbQLNQ/image.png)](https://postimg.cc/sQZ25zSw)

## *需注意事项：

### 0x01

整个过程有几点需要额外注意，很多恶意软件会将服务伪装成svchost.exe，用来迷惑用户。修改注册表有风险，如果在
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Image File Execution Options
键下添加svchost.exe或者系统其他进程如winlog.exe等，在计算机重启时会遇到麻烦。原因很简单，系统是对可执行文件的名称进行处理的，所以在真正的svchost.exe被执行时系统也会尝试将其加载到OD/Windbg进行调试。所以一个好的解决方法就是将服务设置为手动开启，然后修改文件名，并且修改对应服务的路径中可执行文件的名称。所以就理解为什么我调试的文件名称是scvhost.exe而不是svchost.exe。
### 0x02
使用尽量使用OD的原始版本，在跳出窗口时带有各种插件的OD会对系统造成影响，导致黑屏打不开。Windbg可以正常打开。
