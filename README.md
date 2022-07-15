# octoprint-box

Octoprint 安装

盒子方面，我用1G/4G的配置也可以用。推荐华为悦盒的电视盒子（重点感谢神雕的白嫖精神输出https://www.histb.com）。我这里是用华为悦盒ec6108v9，海思hi3798mv100的CPU，mv100系列有wifi驱动，另外要自己打上串口驱动，这里不详细说，盒子刷系统打驱动补丁已经有很多具体教程。我自己测试过，其实用orangepi　lite的配置是不够用的，摄像头延时很严重，真的不如这种垃圾盒子好用。

这里默认你有一个LINUX系统的盒子，有root权限，有wifi（你用有线网络也可以），固定了IP。同时支持USB转串口芯片的驱动还有支持了USB摄像头。

安装前，先测试linux是否有usb转串口的驱动，一般是ch340 ch341，3d打印机上电接上盒子。
终端输入ls /dev/ttyUSB* 看看有没有输出就知道。如果要摄像头，也接上，输入ls /dev/video*看有没有输出。

先来个标准操作
cd ~
sudo apt update
创建个PI用户

sudo adduser pi
sudo adduser pi sudo
sudo visudo
增加pi ALL=(ALL) NOPASSWD:ALL

sudo usermod -a -G tty pi 
sudo usermod -a -G dialout pi
sudo usermod -a -G video pi

sudo su pi
cd~
sudo apt install python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev build-essential
mkdir OctoPrint && cd OctoPrint
python3 -m venv venv
source venv/bin/activate
pip install pip --upgrade
pip install octoprint
如果有报错，就再安装一次。
octoprint serve
这里访问IP:5000就出来了。不过后面我们会改到8111，方便统一打通摄像头端口。特别外网访问回家监控的时候就知道好处了。



octoprint自动启动脚本
wget https://github.com/OctoPrint/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service

sudo systemctl enable octoprint.service.

Webcam安装:
cd ~
sudo apt install subversion libjpeg62-turbo-dev imagemagick ffmpeg libv4l-dev cmake
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
export LD_LIBRARY_PATH=.
make

报错，缺个cmake，综合教程和实际报错，先打上
sudo apt-get install subversion libjpeg8-dev imagemagick libv4l-dev cmake
再make
测试：
./mjpg_streamer -i "./input_uvc.so" -o “./output_http.so"
会显示能操作。

vi创建在/home/pi/scripts目录下webcamDaemon文件，其实你直接把现成的文件拉到对应目录也行。
mkdir /home/pi/scripts
sudo vi /home/pi/scripts/webcamDaemon
要注意，不要被吞字了。检查下开头。

#!/bin/bash
MJPGSTREAMER_HOME=/home/pi/mjpg-streamer/mjpg-streamer-experimental
MJPGSTREAMER_INPUT_USB="input_uvc.so"
MJPGSTREAMER_INPUT_RASPICAM="input_raspicam.so"

#. init configuration
camera="auto"
camera_usb_options="-d /dev/video0 -r 1280x720 -f 10"
camera_raspi_options="-fps 10"

if [ -e "/boot/octopi.txt" ]; then
    source "/boot/octopi.txt"
fi

#. runs MJPG Streamer, using the provided input plugin + configuration
function runMjpgStreamer {
    input=$1
    pushd $MJPGSTREAMER_HOME
    echo Running ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    LD_LIBRARY_PATH=. ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    popd
}

#. starts up the RasPiCam
function startRaspi {
    logger "Starting Raspberry Pi camera"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_RASPICAM $camera_raspi_options"
}

# starts up the USB webcam
function startUsb {
    logger "Starting USB webcam"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_USB $camera_usb_options"
}

# we need this to prevent the later calls to vcgencmd from blocking
# I have no idea why, but that's how it is...
vcgencmd version

#. echo configuration
echo camera: $camera
echo usb options: $camera_usb_options
echo raspi options: $camera_raspi_options

#. keep mjpg streamer running if some camera is attached
while true; do
    if [ -e "/dev/video0" ] && { [ "$camera" = "auto" ] || [ "$camera" = "usb" ] ; }; then
        startUsb
    elif [ "`vcgencmd get_camera`" = "supported=1 detected=1" ] && { [ "$camera" = "auto" ] || [ "$camera" = "raspi" ] ; }; then
        startRaspi
    fi

    sleep 120
done

chmod +x /home/pi/scripts/webcamDaemon
－－－－－－－－－－

在/boot目录下，vi创建octopi.txt
要注意，不要被吞字了。检查下开头。

### Do not use Notepad or WordPad.

### MacOSX users: If you use Textedit to edit this file make sure to use 
### "plain text format" and "disable smart quotes" in "Textedit > Preferences"

### Configure which camera to use
#
#. Available options are:
#. - auto: tries first usb webcam, if that's not available tries raspi cam
#. - usb: only tries usb webcam
#. - raspi: only tries raspi cam
#.
#. Defaults to auto
#.
camera="usb"

### Additional options to supply to MJPG Streamer for the USB camera
#.
#. See https://faq.octoprint.org/mjpg-streamer-config for available options
#.
#. Defaults to a resolution of 640x480 px and a framerate of 10 fps
#
camera_usb_options="-d /dev/video0 -r 1280x720 -f 10"

### Additional webcam devices known to cause problems with -f
#.
#. Apparently there a some devices out there that with the current 
#. mjpg_streamer release do not support the -f parameter (for specifying 
#. the capturing framerate) and will just refuse to output an image if it 
#. .is supplied.
#.
#. The webcam daemon will detect those devices by their USB Vendor and Product
#. ID and remove the -f parameter from the options provided to mjpg_streamer.
#
#. .By default, this is done for the following devices:
#.   Logitech C170 (046d:082b)
#.   GEMBIRD (1908:2310)
#.   Genius F100 (0458:708c)
#.   Cubeternet GL-UPC822 UVC WebCam (1e4e:0102)
#
#Using the following option it is possible to add additional devices. If
#your webcam happens to show above symptoms, try determining your cam's
#vendor and product id via lsusb, activating the line below by removing # and 
#adding it, e.g. for two broken cameras "aabb:ccdd" and "aabb:eeff"
#.
#.   additional_brokenfps_usb_devices=("aabb:ccdd" "aabb:eeff")
#.
#If this fixes your problem, please report it back so we can include the device
#out of the box: https://github.com/guysoft/OctoPi/issues
#
additional_brokenfps_usb_devices=(046d:082d)

### Additional options to supply to MJPG Streamer for the RasPi Cam
#.
#See https://faq.octoprint.org/mjpg-streamer-config for available options
#.
#.Defaults to 10fps
#.
#camera_raspi_options="-fps 10"

### Configuration of camera HTTP output
#.
#. Usually you should NOT need to change this at all! Only touch if you
#. .know what you are doing and what the parameters mean.
#.
#Below settings are used in the mjpg-streamer call like this:
#.
#,   -o "output_http.so -w $camera_http_webroot $camera_http_options"
#.
#Current working directory is the mjpg-streamer base directory.
#.
camera_http_webroot="./www"
camera_http_options=“”


——————
在/etc/systemd/system,vi创建webcam.service
sudo vi /etc/systemd/system/webcam.service
要注意，不要被吞字了。检查下开头。
[Unit]
Description=Camera streamer for OctoPrint
After=network-online.target OctoPrint.service
Wants=network-online.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/scripts/webcamDaemon

[Install]
WantedBy=multi-user.target


最后
sudo systemctl daemon-reload
sudo systemctl enable webcam.service
———-

安装Haproxy


sudo apt-get install haproxy

sudo vi /etc/haproxy/haproxy.cfg

增加：



frontend public
        bind *:8111
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        http-request replace-path /webcam/(.*)   /\1
        server webcam1  127.0.0.1:8080

启动haproxy
~#/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg 


systemctl daemon-reload
systemctl enable haproxy.service
systemctl start haproxy.service



重启前，还有一个，开放摄像头的权限
sudo chmod +x /dev/video0

最后
sudo hostnamectl set-hostname octopi
sudo reboot

以后访问，用8111端口,octopi.local:81111

octoprint里面摄像头地址配置。
Stream URL: /webcam/?action=stream
Snapshot URL: http://127.0.0.1:8080/?action=snapshot
Path to FFMPEG: /usr/bin/ffmpeg


当你走投无路的时候，试试重启一次。其余没什么要特别的，遇到问题就搜索解决问题。


资料来源：
https://community.octoprint.org/t/setting-up-octoprint-on-a-raspberry-pi-running-raspberry-pi-os-debian/2337
https://3dprintscape.com/install-octoprint-on-linux/


神雕系统打wifi驱动：
https://bbs.histb.com/d/18-wifi
打USB转串口驱动：
复制ch341.ko到/lib/modules/#uname -r/目录，然后sudo insmod ch341.ko

