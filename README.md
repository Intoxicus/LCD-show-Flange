on Debian 13 it copies the config to 99-fbturbo.~
this name is not a .conf, so X ignores it
at the same time, /etc/X11/xorg.conf.d/99-v3d.conf continues to force the modesetting path
who wants a fix here it is for 3.5 display:

#!/bin/bash

sudo ./system_backup.sh

if [ -f /etc/X11/xorg.conf.d/40-libinput.conf ]; then
sudo rm -rf /etc/X11/xorg.conf.d/40-libinput.conf
fi
if [ ! -d /etc/X11/xorg.conf.d ]; then
sudo mkdir -p /etc/X11/xorg.conf.d
fi
sudo cp ./usr/tft35a-overlay.dtb /boot/overlays/
sudo cp ./usr/tft35a-overlay.dtb /boot/overlays/tft35a.dtbo
#root_dev=`grep -oPr "root=[^\s]*" /boot/cmdline.txt | awk -F= '{printf $NF}'`
#if test "$root_dev" = "/dev/mmcblk0p7";then
#sudo cp -rf ./boot/config-noobs-nomal.txt ./boot/config.txt.bak
#else
#sudo cp -rf ./boot/config-nomal.txt ./boot/config.txt.bak
#sudo echo "hdmi_force_hotplug=1" >> ./boot/config.txt.bak
#fi

source ./system_config.sh
#sudo sed -i -e 's/dtoverlay=vc4-kms-v3d/#dtoverlay=vc4-kms-v3d/' ./boot/config.txt.bak
sudo echo "hdmi_force_hotplug=1" >> ./boot/config.txt.bak
sudo echo "dtparam=i2c_arm=on" >> ./boot/config.txt.bak
sudo echo "dtparam=spi=on" >> ./boot/config.txt.bak
sudo echo "enable_uart=1" >> ./boot/config.txt.bak
sudo echo "dtoverlay=tft35a:rotate=90" >> ./boot/config.txt.bak
sudo echo "hdmi_group=2" >> ./boot/config.txt.bak
sudo echo "hdmi_mode=1" >> ./boot/config.txt.bak
sudo echo "hdmi_mode=87" >> ./boot/config.txt.bak
sudo echo "hdmi_cvt 480 320 60 6 0 0 0" >> ./boot/config.txt.bak
sudo echo "hdmi_drive=2" >> ./boot/config.txt.bak
sudo cp -rf ./boot/config.txt.bak /boot/config.txt

sudo cp -rf ./usr/99-calibration.conf-35-90  /etc/X11/xorg.conf.d/99-calibration.conf
if [[ "$deb_version" < "12.1" ]]; then
sudo cp -rf ./usr/99-fbturbo.conf  /usr/share/X11/xorg.conf.d/99-fbturbo.conf
fi

if [[ "$deb_version" = "13.1" ]] || [[ "$deb_version" > "13.1" ]]; then
sudo cp -rf ./etc/.bash_profile /home/$username/
mkdir -p /home/$username/.config/xorg.conf.d
cat <<'EOF' > /home/$username/.config/xorg.conf.d/99-fbdev.conf
Section "Device"
        Identifier      "SPI LCD FBDEV"
        Driver          "fbdev"
        Option          "fbdev" "/dev/fb1"
EndSection
EOF
cat <<'EOF' > /home/$username/.xserverrc
#!/bin/sh
exec /usr/bin/X -configdir /home/pi/.config/xorg.conf.d -nolisten tcp "$@"
EOF
chmod 755 /home/$username/.xserverrc

#sudo cp -rf ./etc/rc_x11.local /etc/rc.local
fi

#if test "$root_dev" = "/dev/mmcblk0p7";then
#sudo cp ./usr/cmdline.txt-noobs /boot/cmdline.txt
#else
#sudo cp ./usr/cmdline.txt /boot/
#fi
if [[ "$deb_version" < "13.1" ]]; then
sudo cp ./usr/inittab /etc/
fi
#sudo cp ./boot/config-35.txt /boot/config.txt
sudo touch ./.have_installed
echo "gpio:resistance:35:90:480:320" > ./.have_installed

if [[ "$deb_version" < "13.1" ]]; then
if [[ "$deb_version" < "12.10" ]]; then
sudo apt-get update
fi
#FBCP install
wget --spider -q -o /dev/null --tries=1 -T 10 https://cmake.org/
if [ $? -eq 0 ]; then
#sudo apt-get update
sudo apt-get install cmake libraspberrypi-dev -y 2> error_output.txt
result=`cat ./error_output.txt`
echo -e "\033[31m$result\033[0m"
grep -q "^E:" ./error_output.txt
type cmake > /dev/null 2>&1
if [ $? -eq 0 ]; then
sudo rm -rf rpi-fbcp
wget --spider -q -o /dev/null --tries=1 -T 10 https://github.com
if [ $? -eq 0 ]; then
sudo git clone https://github.com/tasanakorn/rpi-fbcp
if [ $? -ne 0 ]; then
echo "download fbcp failed, copy native fbcp!!!"
sudo cp -r ./usr/rpi-fbcp .
fi
else
echo "bad network, copy native fbcp!!!"
sudo cp -r ./usr/rpi-fbcp .
fi
sudo mkdir ./rpi-fbcp/build
cd ./rpi-fbcp/build/
sudo cmake ..
sudo make
sudo install fbcp /usr/local/bin/fbcp
cd - > /dev/null
type fbcp > /dev/null 2>&1
if [ $? -eq 0 ]; then
if [[ "$deb_version" < "12.1" ]]; then
sudo cp -rf ./usr/99-fbturbo-fbcp.conf  /usr/share/X11/xorg.conf.d/99-fbturbo.conf
fi
sudo cp -rf ./etc/rc.local /etc/rc.local
fi
else
echo "install cmake error!!!!"
fi
else
echo "bad network, can't install cmake!!!"
fi
fi

#evdev install
#nodeplatform=`uname -n`
#kernel=`uname -r`
version=`uname -v`
#if test "$nodeplatform" = "raspberrypi";then
#echo "this is raspberrypi kernel"
input_result=0
version=${version##*(}
version=${version%%-*} 
echo $version
if test $version -lt 2017;then
echo "reboot"
else
echo "need to update touch configuration"
wget --spider -q -o /dev/null --tries=1 -T 10 http://mirrors.zju.edu.cn/raspbian/raspbian
if [ $? -ne 0 ]; then
input_result=1
else
sudo apt-get install xserver-xorg-input-evdev  2> error_output.txt
dpkg -l | grep xserver-xorg-input-evdev > /dev/null 2>&1
if [ $? -ne 0 ]; then
input_result=1
fi
fi
if [ $input_result -eq 1 ]; then 
if [ $hardware_arch -eq 32 ]; then
sudo dpkg -i -B ./xserver-xorg-input-evdev_1%3a2.10.6-1+b1_armhf.deb 2> error_output.txt
elif [ $hardware_arch -eq 64 ]; then
sudo dpkg -i -B ./xserver-xorg-input-evdev_1%3a2.10.6-2_arm64.deb 2> error_output.txt
fi
fi
result=`cat ./error_output.txt`
echo -e "\033[31m$result\033[0m"
grep -q "error:" ./error_output.txt && exit
sudo cp -rf /usr/share/X11/xorg.conf.d/10-evdev.conf /usr/share/X11/xorg.conf.d/45-evdev.conf
#echo "reboot"
fi
#else
#echo "this is not raspberrypi kernel, no need to update touch configure, reboot"
#fi

sudo sync
sudo sync
sleep 1
if [ $# -eq 1 ]; then
sudo ./rotate.sh $1
elif [ $# -gt 1 ]; then
echo "Too many parameters"
fi

echo "reboot now"
sudo reboot



### Install drivers in the Ubuntu system
https://github.com/lcdwiki/LCD-show-ubuntu

### Install drivers in the Kali system
https://github.com/lcdwiki/LCD-show-kali

### Install drivers in the RetroPie system
https://github.com/lcdwiki/LCD-show-retropie



Install drivers in the Raspbian system<br>
====================================================
Update: <br>
  v2.1-20191106<br>
  Update to support MHS35B<br>
Update: <br>
  v2.0-20190704<br>
  Update to support rotate the display direction<br>
Update: <br>
  v1.9-20181204<br>
  Update to support MHS40 & MHS32<br>
Update: <br>
  v1.8-20180907<br>
  Update to support MHS35<br>
Update: <br>
  v1.7-20180320<br>
  Update to support Raspbian Version: March 2018(Release date:2018-03-13)<br>
Update: <br>
  v1.6-20170824<br>
  Update xserver to support Raspbian-2017-08-16<br>
Update: <br>
  v1.5-20170706<br>
  Update to support Raspbian-2017-07-05, Raspbian-2017-06-21<br>
Update: <br>
  v1.3-20170612<br>
  fixed to support Raspbian-2017-03-02, Raspbian-2017-04-10<br>
Update: <br>
  v1.2-20170302<br>
  Add xserver-xorg-input-evdev_1%3a2.10.3-1_armhf.deb to support Raspbian-2017-03-02<br>
Update: <br>
  v1.1-20160815<br><br>


# How to install the LCD driver of Raspberry Pi
  
1.)Step1, Install Raspbian official mirror <br>
====================================================
  a)Download Raspbian official mirror:<br>
  https://www.raspberrypi.org/downloads/<br>
  b)Use“SDFormatter.exe”to Format your TF Card<br>
  c)Use“Win32DiskImager.exe” Burning mirror to TF Card<br>
     
2.) Step2, Clone my repo onto your pi<br>
====================================================
Use SSH to connect the Raspberry Pi, <br>
And Ensure that the Raspberry Pi is connected to the Internet before executing the following commands:
-----------------------------------------------------------------------------------------------------

```sudo rm -rf LCD-show```<br>
```git clone https://github.com/goodtft/LCD-show.git```<br>
```chmod -R 755 LCD-show```<br>
```cd LCD-show/```<br>
  
3.)Step3, According to your LCD's type, excute the corresponding driver:
====================================================

# 2.4” RPi Display (MPI2401):
### Driver install:
sudo ./LCD24-show
### WIKI:
CN: http://www.lcdwiki.com/zh/2.4inch_RPi_Display  <br>
EN: http://www.lcdwiki.com/2.4inch_RPi_Display
 

# 2.4” RPi Display For RPi 3A+ (MPI2411):
### Driver install:
sudo ./LCD24-3A+-show  
### WIKI:
CN: http://www.lcdwiki.com/zh/2.4inch_RPi_Display_For_RPi_3A+   <br>
EN: http://www.lcdwiki.com/2.4inch_RPi_Display_For_RPi_3A+

# 2.8” RPi Display (MPI2801):
### Driver install:
sudo ./LCD28-show 
### WIKI:
CN: http://www.lcdwiki.com/zh/2.8inch_RPi_Display  <br>
EN: http://www.lcdwiki.com/2.8inch_RPi_Display
  
# 3.2” RPi Display (MPI3201):
### Driver install:
sudo ./LCD32-show   
### WIKI:
CN: http://www.lcdwiki.com/zh/3.2inch_RPi_Display  <br>
EN: http://www.lcdwiki.com/3.2inch_RPi_Display

# MHS-3.2” RPi Display (MHS3232):
### Driver install:
sudo ./MHS32-show   
### WIKI:
CN: http://www.lcdwiki.com/zh/MHS-3.2inch_Display  <br>
EN: http://www.lcdwiki.com/MHS-3.2inch_Display

# 3.5” RPi Display(MPI3501):
### Driver install:
sudo ./LCD35-show
### WIKI:
CN: http://www.lcdwiki.com/zh/3.5inch_RPi_Display  <br>
EN: http://www.lcdwiki.com/3.5inch_RPi_Display
   
# 3.5” HDMI Display-B(MPI3508):
### Driver install:
sudo ./MPI3508-show
### WIKI:
CN: http://www.lcdwiki.com/zh/3.5inch_HDMI_Display-B  <br>
EN: http://www.lcdwiki.com/3.5inch_HDMI_Display-B
    
# MHS-3.5” RPi Display(MHS3528):
### Driver install:
sudo ./MHS35-show
### WIKI:
CN: http://www.lcdwiki.com/zh/MHS-3.5inch_RPi_Display  <br>
EN:http://www.lcdwiki.com/MHS-3.5inch_RPi_Display

# MHS-3.5” RPi Display-B(MHS35XX):
### Driver install:
sudo ./MHS35B-show
### WIKI:
CN: http://www.lcdwiki.com/zh/MHS-3.5inch_RPi_Display-B  <br>
EN:http://www.lcdwiki.com/MHS-3.5inch_RPi_Display-B

# 4.0" HDMI Display(MPI4008):
### Driver install:
sudo ./MPI4008-show
### WIKI:
CN: http://www.lcdwiki.com/zh/4inch_HDMI_Display-C  <br>
EN: http://www.lcdwiki.com/4inch_HDMI_Display-C
   
# MHS-4.0" HDMI Display-B(MHS4001):
### Driver install:
sudo ./MHS40-show
### WIKI:
CN: http://www.lcdwiki.com/zh/MHS-4.0inch_Display-B  <br>
EN: http://www.lcdwiki.com/MHS-4.0inch_Display-B
  
# 5.0” HDMI Display(Resistance touch)(MPI5008):
### Driver install:
sudo ./LCD5-show
### WIKI:
CN: http://www.lcdwiki.com/zh/5inch_HDMI_Display  <br>
EN: http://www.lcdwiki.com/5inch_HDMI_Display
    
# 5inch HDMI Display-B(Capacitor touch)(MPI5001):
### Driver install:
sudo ./MPI5001-show
### WIKI:
CN: http://www.lcdwiki.com/zh/5inch_HDMI_Display-B  <br>
EN: http://www.lcdwiki.com/5inch_HDMI_Display-B
    
# 7inch HDMI Display-B-800X480(MPI7001):
### Driver install:
sudo ./LCD7B-show
### WIKI:
CN: http://www.lcdwiki.com/zh/7inch_HDMI_Display-B  <br>
EN: http://www.lcdwiki.com/7inch_HDMI_Display-B
   
# 7inch HDMI Display-C-1024X600(MPI7002):
### Driver install:
sudo ./LCD7C-show
### WIKI:
CN: http://www.lcdwiki.com/zh/7inch_HDMI_Display-C  <br>
EN: http://www.lcdwiki.com/7inch_HDMI_Display-C
   
Wait for a moment after executing the above command, then you can use the corresponding raspberry LCD.




# How to rotate the display direction

This method only applies to the Raspberry Pi series of display screens, other display screens do not apply.

### Method 1, If the driver is not installed, execute the following command (Raspberry Pi needs to connected to the Internet):

sudo rm -rf LCD-show<br>
git clone https://github.com/goodtft/LCD-show.git<br>
chmod -R 755 LCD-show<br>
cd LCD-show/<br>
sudo ./XXX-show 90<br>

After execution, the driver will be installed. The system will automatically restart, and the display screen will rotate 90 degrees to display and touch normally.<br>
( ' XXX-show ' can be changed to the corresponding driver, and ' 90 ' can be changed to 0, 90, 180 and 270, respectively representing rotation angles of 0 degrees, 90 degrees, 180 degrees, 270 degrees)<br>

### Method 2, If the driver is already installed, execute the following command:

cd LCD-show/<br>
sudo ./rotate.sh 90<br>

After execution, the system will automatically restart, and the display screen will rotate 90 degrees to display and touch normally.<br>
( ' 90 ' can be changed to 0, 90, 180 and 270, respectively representing rotation angles of 0 degrees, 90 degrees, 180 degrees, 270 degrees)<br>
(If the rotate.sh prompt cannot be found, use Method 1 to install the latest drivers)



