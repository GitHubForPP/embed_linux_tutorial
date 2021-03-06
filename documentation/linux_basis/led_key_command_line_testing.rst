.. vim: syntax=rst

使用命令点灯和检测按键
-----------

前面讲了那么多的内容，终于迎来了重头戏——使用开发板的各种硬件外设，如入门必备的点灯，按键，蜂鸣器，串口等等。本章先带领大家使用命令行来控制LED灯和检测按键。

初探/sys目录
~~~~~~~~

与/proc目录类似，/sys目录下的文件/文件夹向用户提供了一些关于设备、内核模块、文件系统以及其他内核组件的信息，如子目录block中存放了所有的块设备；子目录bus中存放了系统中所有的总线类型，有i2c、usb、sdio、pci等；子目录class按类型归类设备，如leds、lcd、mtd、p
wm等。

尝试在开发板的终端执行以下命令查看sys各层级的目录内容：

#在开发板上的终端执行以下命令查看

ls /sys

ls /sys/class

ls /sys/class/leds

ls /sys/class/leds/cpu

|ledkey002|

图 13‑1 /sys各目录的内容（内容可能会小有差异）

可以看到，本开发板的/sys/class/leds目录下分别有：blue、cpu、green、red等目录（出厂的），它们代表着开发板上不同的LED灯，是由Linux内核驱动提供给用户的控制接口。

以其中的/sys/class/leds/cpu目录为例，它下面又包含brightness、device、max_brightness、power、subsystem、trigger、uevent等文件。其中brightness表示LED灯的亮度，trigger表示LED灯的触发方式，我们可以通过ec
ho、cat等命令来修改或查看这些文件，从而达到控制LED灯的目的。下面以实例进行讲解。

控制LED灯
~~~~~~

在野火的i.MX 6ULL Pro及Mini开发板上，有如下几个LED灯可以通过主控芯片的引脚进行控制，硬件的具体位置见《第10章 野火EBF 6ULL 开发板硬件资源介绍》章节的资源图：

-  心跳灯：默认用来指示系统运行状态，系统正在运行时就会持续闪烁。开发板文件系统上的/sys/class/leds/cpu目录表示的就是该心跳灯。

-  三盏普通LED灯：在Pro开发板上，它们共同组成一盏RGB彩灯，可以进行色彩混合。在Mini开发板上是三盏独立的小灯。开发板文件系统上的/sys/class/leds/red、/sys/class/leds/blue、/sys/class/leds/green目录表示的就是这三盏灯。

brightness文件
^^^^^^^^^^^^

各LED灯设备下的brightness文件表示了它的亮度值。本开发板提供的内核驱动中，心跳灯是直接IO控制的，它的亮度范围为1和0表示亮灭；而其它三盏普通LED灯是使用pwm控制的，支持的亮度范围为0~255。

我们先尝试控制三盏普通LED灯，在开发板的终端上执行如下命令，执行前请确认当前为root用户：

#在后面的说明中，#号表示注释，它后面的内容不要输入到终端中

#LED灯默认可能处于亮的状态，我们先把它们全部关闭再一盏盏点亮

echo 0 > /sys/class/leds/red/brightness #关闭red灯

echo 0 > /sys/class/leds/blue/brightness #关闭blue灯

echo 0 > /sys/class/leds/green/brightness #关闭green灯

echo 255 > /sys/class/leds/red/brightness #点亮red灯

echo 255 > /sys/class/leds/blue/brightness #点亮blue灯

echo 255 > /sys/class/leds/green/brightness #点亮green灯

|ledkey003|

图 13‑2 完全关闭RGB灯后，点亮GREEN灯

在Pro开发板中，由于三盏灯会进行硬件RGB混色，所以随着每条指令的执行，彩灯的颜色不一定呈红蓝绿的变化，而是三盏灯状态的共同结果，可以自行尝试使用不同的亮度值对RGB灯进行混色。

在Mini开发板上三盏LED是硬件独立的，可以更明显地看出执行不同指令时硬件的运行情况。

trigger文件
^^^^^^^^^

如果我们直接使用上面的方式去控制/sys/class/leds/cpu心跳灯的brightness文件，发现并没有生效，心跳灯依然是按自己的节奏在闪烁，这是因为我们的驱动中心跳灯与上述普通LED灯使用了不同的默认触发条件。LED灯的触发条件可以通过其目录下的trigger文件控制。

我们先来查看各LED设备trigger文件的原内容：

#在后面的说明中，#号表示注释，它后面的内容不要输入到终端中

cat /sys/class/leds/red/trigger #查看red灯的trigger文件内容

cat /sys/class/leds/cpu/trigger #查看cpu灯的trigger文件内容

|ledkey004|

图 13‑3 各LED灯的默认trigger配置

仔细对比图 13‑3中red及cpu灯的trigger文件内容，发现两个文件中有类似的内容，这些内容表示触发条件，其中“[]”括号括起来的表示当前使用的触发方式。可看到red灯中的触发条件为“none”，表示无触发条件，所以直接修改brightness文件来控制；而cpu灯的触发条件为“heartb
eat”，表示系统心跳指示触发，这时用户不能再直接修改brightness文件来控制。其它的触发条件如nand-disk表示nand读写指示灯，timer表示通过定时器控制等。

了解触发条件后，我们可以先把cpu灯的触发条件改为none，再修改brightness文件即可以手动控制cpu灯，尝试执行如下命令：

#在后面的说明中，#号表示注释，它后面的内容不要输入到终端中

echo none > /sys/class/leds/cpu/trigger #修改cpu灯触发条件为none

cat /sys/class/leds/cpu/trigger #查看cpu灯的trigger文件内容

echo 1 > /sys/class/leds/cpu/brightness # 点亮cpu灯

echo 0 > /sys/class/leds/cpu/brightness #关闭cpu灯

|ledkey005|

图 13‑4 修改cpu灯的触发条件并控制

类似地，我们可以把red灯的触发条件改为heartbeat，此时red灯就会以心跳指示的方式进行闪烁：

echo heartbeat > /sys/class/leds/red/trigger

|ledkey006|

图 13‑5 把red灯改为heartbeat触发

在示例中我们一直是用echo命令修改文件内容来达到控制的目的，但我们要清楚地知道，并不是echo命令能控制硬件，而是文件的内容影响了硬件的工作状态。可以尝试使用Vi编辑器来修改各个LED灯的brightness文件的内容，完全可以达到同样的控制目的。不过使用Vi时可能会提示权限问题，忽略即可。

初探/dev目录
~~~~~~~~

除了/proc和/sys目录外，/dev目录也包含了非常丰富的设备信息，该目录下包含了Linux系统中使用的所有外部设备，如/dev/tty为串口设备、/dev/ram为内存、通过这些设备文件，我们也可以访问到对应的硬件设备。

尝试使用以下命令查看dev目录的内容：

ls /dev

ls /dev/input

|ledkey007|

图 13‑6 /dev目录的部分内容

以图 13‑6中/dev/input目录下的内容为例，上面包含了event0/1/2/3等内容，它们是输入设备的事件文件接口，通过它们可以获知设备上报的输入事件。event后带的数字并不是跟设备绑定的，可以查看文件/proc/bus/input/devices了解它们分别代表什么。

在开发板的终端执行以下命令：

cat /proc/bus/input/devices

|ledkey008|

图 13‑7 查看输入设备（具体内容可能有差异）

如图 13‑7，可看到当前开发板的包含了四个输入设备，在具体的开发板上可能会有差异：

-  event0：名称为20cc000.snvs:snvs-powerkey，它对应的硬件是开发板上的ON/OFF按键。

-  event1：名称为goodix-ts，它对应的是触摸屏设备，若没接入触摸屏可能会见不到该设备

-  event2：名称为gpio_ir_recv，它对应的是红外接收器，我们提供的部分内核驱动可能不包含红外接收器，该设备可能不存在。

-  event3：名称为gpio_keys，它对应的是KEY按键。

检测按键
~~~~

在开发板的文件系统中我们提供了evtest工具，使用它可以更方便地查看硬件当前接入的输入设备，并且对其进行检测。

在开发板的终端上输入如下命令：

evtest

|ledkey009|

图 13‑8 evtest查看可用的输入设备（视具体的接入设备有差异）

执行命令后，它会扫描/dev/input目录下的event设备输入事件文件，并列出到终端，它提示我们可以通过数字选择对应的设备进行测试，具体请根据自己开发板上的输出进行选择。此处我们以snvs-powerkey和gpio-keys为例，测试结果如图 13‑9和图
13‑10所示。测试时可执行“Ctrl”+“c”退出。

|ledkey010|

图 13‑9 ON/OFF按键的测试

|ledkey011|

图 13‑10 KEY按键的测试

设备、文件与驱动
~~~~~~~~

通过前面使用LED灯和按键的控制过程，可体会到Linux系统中设备与文件的联系。控制时我们根本不需要知道LED灯和按键的具体硬件连接，只要知道哪个文件代表哪个设备，然后就可以通过文件以同样的方式操控同类设备了。这就是驱动的作用，它把硬件设备与Linux文件建立了映射关系。

在MCU的裸机程序中，要控制LED灯和按键，需要先了解它们与MCU硬件是如何连接的，即要知道连接的具体的IO引脚，然后修改寄存器配置IO的输入输出，工作模式等，最后编写相关的控制函数API提供给上层调用。编写应用时用户通过函数API去控制硬件。

在Linux系统下，驱动工程师的工作其实类似，他们也需要了解LED或按键是如何与CPU连接的，也得配置好IO的工作模式，区别在于还需要按照Linux系统的接口要求编写相关控制函数提供给Linux的虚拟文件系统，使得硬件与文件建立起联系，提供给Linux用户。编写应用时用户通过设备文件即可控制硬件。

所以说，驱动工程师所做的工作就是把硬件的控制方式抽象、封装成为函数API或者文件，只不过Linux驱动工程师在封装时需要按照Linux的规范去编写，而且在编写时需要了解Linux内核的一些机制，所以相对裸机驱动来说，提出了更高的要求。

.. |ledkey002| image:: media/ledkey002.png
   :width: 5.76806in
   :height: 1.22371in
.. |ledkey003| image:: media/ledkey003.png
   :width: 5.17361in
   :height: 1.22533in
.. |ledkey004| image:: media/ledkey004.png
   :width: 5.45047in
   :height: 1.18344in
.. |ledkey005| image:: media/ledkey005.png
   :width: 5.09028in
   :height: 1.12159in
.. |ledkey006| image:: media/ledkey006.png
   :width: 5.51714in
   :height: 0.65006in
.. |ledkey007| image:: media/ledkey007.png
   :width: 5.76806in
   :height: 1.82054in
.. |ledkey008| image:: media/ledkey008.png
   :width: 5.625in
   :height: 4.85091in
.. |ledkey009| image:: media/ledkey009.png
   :width: 4.29167in
   :height: 1.31409in
.. |ledkey010| image:: media/ledkey010.png
   :width: 5.18056in
   :height: 3.00428in
.. |ledkey011| image:: media/ledkey011.png
   :width: 5.32639in
   :height: 3.18093in
