# Guide to Cross Compile Qt 5.9.8 for RaspberryPi3

This guide is heavly based on the information presented [here](https://wiki.qt.io/RaspberryPi2EGLFS).
I will be distilling this information down to the exact steps I took to get cross compiling working.

## Setup as of 5/7/2019  
Host computer: Ubuntu 18.04 64bit  
Qt Version: 5.9  
Pi: Raspberry Pi3 on Rasbian Jessie img from 2017-07-05  
**It is very important that the Pi is on Rasbian Jessie. Any other OS or version of Rasbian, will result in cross compliaiton likley failing.**


I will assume that the host computer and the pi OSes will be freshly installed and the OSes used are those listed above.

## Prepare the Host:
**1. Update packages**  
```
sudo apt-get update
sudo apt-get ugrade
```

**2. Install still missing packages**
```
sudo apt-get install build-essential
sudo apt-get install git
sudo apt-get install python
```

**3. Prepare cross compile workspace and grab toolchain and Qt 5.9**
```
mkdir ~/raspi
cd ~/raspi
git clone https://github.com/raspberrypi/tools
git clone git://code.qt.io/qt/qtbase.git -b 5.9
```

## Preparing the Pi:
**1. Grab FW from clean install**
```
sudo rpi-update
reboot
```

**2. Edit sources**  
Open the file list below and un-comment the line starting with deb-src
```
sudo nano /etc/apt/sources.list
```

**3. Update Packages**
```
sudo apt-get update
sudo apt-get ugrade
```

**4. Install Qt Dependencies**
```
sudo apt-get update
sudo apt-get build-dep qt4-x11
sudo apt-get build-dep libqt5gui5
sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0
```
*If you cannot find one of these packages it is most likely becase the sources.list file was not properly edited. Make sure to re-check step 2 of Preparing the Pi.*  

**5. Prepare Target Directory**
```
sudo mkdir /usr/local/qt5pi
sudo chown pi:pi /usr/local/qt5pi
```
*Where pi in "pi:pi" is the username on the pi*


## Cross Compiling from Host Computer:
**1. Creating sys root**  
This step takes the libaries added to the the pi and pulls them to our host computer to help create our cross compiler monstrocity.  
Below, pi is the username on the raspberry pi and local_ip_address should be replaced by the IP address of the pi on your local netowrk. This IP can be found by running the following command on the Raspberry Pi.
```
ifconfig -a
```

```
mkdir sysroot sysroot/usr sysroot/opt
rsync -avz pi@local_ip_address:/lib sysroot
rsync -avz pi@local_ip_address:/usr/include sysroot/usr
rsync -avz pi@local_ip_address:/usr/lib sysroot/usr
rsync -avz pi@local_ip_address:/opt/vc sysroot/opt
```

Most of the rsync command won't take that long to finish, but the third rsync will take some serious time.
I did have one of the rsync commands tell me that some files were not transfered.I'm not sure which one it was, but it on occurs on one command, and the message occurs every time that command is run. It should not impact your ability to cross compile. 

**2. Adjust links**
```
wget https://raw.githubusercontent.com/Kukkimonsuta/rpi-buildqt/master/scripts/utils/sysroot-relativelinks.py
chmod +x sysroot-relativelinks.py
./sysroot-relativelinks.py sysroot
```

**3. Configure and make Qt**
make sure we are in the cloned Qt repo. There is a bunch of talk about all the possible parmeter for configuration. Below is the one that I used that let me to sucess.
```
./configure -release -opengl es2 -device linux-rasp-pi3-g++ -device-option CROSS_COMPILE=~/raspi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf- -sysroot ~/raspi/sysroot -opensource -confirm-license -make libs -prefix /usr/local/qt5pi -extprefix ~/raspi/qt5pi -hostprefix ~/raspi/qt5 -v -no-use-gold-linker
```  
Once that configuration is sucessful, continue with the make process
```
make
make install
```
Hopefully that this point there are no errors, you have sucessfully cross compiled Qt for the Pi!

## Testting with an example:
**1. Build example**
On the host computer we will build our example and send it to the Pi
```
cd qtbase/examples/opengl/qopenglwidget
~/raspi/qt5/bin/qmake
make
scp qopenglwidget pi@raspberrypi.local:/home/pi
```

**2. Help linker on Pi find Qt libs**
```
echo /usr/local/qt5pi/lib | sudo tee /etc/ld.so.conf.d/00-qt5pi.conf
sudo ldconfig
```

**2. Fix the EGL/GLES libs and symbols**  
I am not sure if this is required or not, but I ran them anyway.
Some of the commands reported that a file already exits at that location, I think that is fine.
```
sudo mv /usr/lib/arm-linux-gnueabihf/libEGL.so.1.0.0 /usr/lib/arm-linux-gnueabihf/libEGL.so.1.0.0_backup
sudo mv /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2.0.0 /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2.0.0_backup
sudo ln -s /opt/vc/lib/libEGL.so /usr/lib/arm-linux-gnueabihf/libEGL.so.1.0.0
sudo ln -s /opt/vc/lib/libGLESv2.so /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2.0.0
sudo ln -s /opt/vc/lib/libbrcmEGL.so /opt/vc/lib/libEGL.so
sudo ln -s /opt/vc/lib/libbrcmGLESv2.so.2 /opt/vc/lib/libGLESv2.so
```
I am not sure if this is required or not, but I ran them anyway.
```
sudo ln -s /opt/vc/lib/libEGL.so /opt/vc/lib/libEGL.so.1
sudo ln -s /opt/vc/lib/libGLESv2.so /opt/vc/lib/libGLESv2.so.2
```

**3. Run the example**
```
cd ~
./qopenglwidget
```

## Adding other Qt Modules:
```
git clone git://code.qt.io/qt/<qt-module>.git -b 5.9
cd <qt-module>

~/raspi/qt5/bin/qmake
make
make install
```
