# octoprint-box

Octoprint 安装

# 操作指南
(https://github.com/waterfish007/octoprint-box/blob/main/安装指南.txt)


盒子方面，我用1G/4G的配置也可以用。推荐华为悦盒的电视盒子（重点感谢神雕的白嫖精神输出https://www.histb.com）。我这里是用华为悦盒ec6108v9，海思hi3798mv100的CPU，mv100系列有wifi驱动，另外要自己打上串口驱动，这里不详细说，盒子刷系统打驱动补丁已经有很多具体教程。我自己测试过，其实用orangepi　lite的配置是不够用的，摄像头延时很严重，真的不如这种垃圾盒子好用。

这里默认你有一个LINUX系统的盒子，有root权限，有wifi（你用有线网络也可以），固定了IP。同时支持USB转串口芯片的驱动还有支持了USB摄像头。

安装前，先测试linux是否有usb转串口的驱动，一般是ch340 ch341，3d打印机上电接上盒子。
终端输入ls /dev/ttyUSB* 看看有没有输出就知道。如果要摄像头，也接上，输入ls /dev/video*看有没有输出。
