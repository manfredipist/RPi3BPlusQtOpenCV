# Cross-Compiled Qt 5.14.2, OpenCV 4.4.0 and RaspiCam 0.1.9 for Raspberry Pi 3

## Disclaimer
This guide is based on the publications of dozens of Raspberry Pi enthusiasts. By the time of writing, i've followed 20 or more different guides thus i will try to credit, as much as i can, all those people whose precious advices helped me achieving this result. Please forgive me if you think i've used your stuff without crediting you, send me an email, i will be more than happy to add you to the credits list.

- https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4
- https://qtcrosscompile.blogspot.com/2020/05/build-cross-compile-qt5.html
- http://thebugfreeblog.blogspot.com/2019/
- https://www.uco.es/investiga/grupos/ava/node/40
- https://www.pyimagesearch.com/2019/09/16/install-opencv-4-on-raspberry-pi-4-and-raspbian-buster/
- https://qengineering.eu/install-opencv-4.2-on-raspberry-pi-4.html

I want to particularly thank @Luca Carlon for having provided a working rpi-toolchain with gcc 8.3, i've tried many of them but only his worked just fine out of the box.

I want also to thank @UvinduW as I'm currently copying his guide template as well as most of his words. I explicity apologize for that but I wanted to provide a working guide without spending too much time on it.

That said, this guide should help you setting up a working environment to build cool stuff on your Raspberry Pi 3B+ using Qt, OpenCV and RaspiCam. Hopefully this guide will help you getting everything up and running without any struggles.

## Step 1: Download and flash the Raspberry Pi OS image

- Download the Raspberry Pi OS image from the Raspbian Foundation. I suggest you to download the lighter version with desktop since we will need some space to build all the libraries we need.
	- Latest version: https://downloads.raspberrypi.org/raspios_armhf_latest
- Flash the image onto an SD card (I used Balena Etcher)
- Setup ssh and wifi connection to access your rpi headless.

### 1.1 Set up SSH
With the SD card connected to your computer, create a new file called `ssh` on the /boot/ partition


### 1.2 Set up WiFi
With the SD card connected to your computer, create a new file called `wpa_supplicant.conf` on the /boot/ partition

The file should have the following contents:

	country=IT
	ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
	update_config=1

	network={
		ssid="NETWORK-SSID"
		psk="NETWORK-PASSWORD"
	}


## Step 2: Configure the Raspberry Pi
Startup your rpi and proceed changing your default password which is 'raspberry' and upgrade the system. Execute the following commands:

	sudo apt-get update
	sudo apt-get upgrade -y
	sudo raspi-config

- Replace your password
- Move to interface options and enable Camera and VNC (optional but strongly suggested for further steps)

Once you've finished with your customization, restart your Raspberry Pi

	sudo reboot -h now

### 2.2 Enable Development Sources
You need to edit your sources list to enable development sources. To do this, enter the following into a terminal

	sudo nano /etc/apt/sources.list

In the nano text editor, uncomment the following line by removing the `#` character (the line should exist already, if not then add it):

	deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi

Now press `Ctrl+X` to quit. You will be asked if you want to save the changes. Press `y` for yes, and then press `Enter` to keep the same filename.

### 2.3 Update the system

Run the following commands in terminal to update the system and reboot

	sudo apt-get update
	sudo apt-get dist-upgrade
	sudo reboot -h now

### 2.4 Enable rsync with elevated rights
Later in this guide, we will be using the `rsync` command to sync files between the PC and the RPi. For some of these files, root rights (i.e. sudo) are required.  
In this step, we will change a setting to allow this.

First, find the path to rsync with the following command:

	which rsync

On my RPi it was here:

	/usr/bin/rsync

Now we need to edit the sudoers file. You can edit it by typing the following into terminal:

	sudo visudo

Now the sudoers file should be opened with nano. You need to add an entry to the end with the following structure:

	<username> ALL=NOPASSWD:<path to rsync>

In my case (and for most others else as well), it was:

	pi ALL=NOPASSWD:/usr/bin/rsync

That's it. Now rsync should be setup to run with sudo if needed.

### 2.5 Install the required development packages

Run the following commands in terminal to install the required packages. In this particular installation i will also install boost libraries, qtlocation and qtmultimedia modules

	sudo apt-get build-dep qt5-qmake
	sudo apt-get build-dep libqt5gui5
	sudo apt-get build-dep libqt5webengine-data
	sudo apt-get build-dep libqt5webkit5
	sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver

	# qtmultimedia
	sudo apt-get install libasound2-dev libpulse-dev gstreamer1.0-omx libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev  gstreamer1.0-alsa

	# qtlocation
	sudo apt-get install libgeoclue-2-dev libdbus-glib-1-dev libgudev-1.0-dev libbluetooth-dev

	# boost libraries and others
	sudo apt-get install libboost1.58-all-dev libudev-dev libinput-dev libts-dev libmtdev-dev libjpeg-dev libfontconfig1-dev libssl-dev libdbus-1-dev libglib2.0-dev libxkbcommon-dev libegl1-mesa-dev libgbm-dev libgles2-mesa-dev mesa-common-dev xcb libxcb-xkb-dev x11-xkb-utils libx11-xcb-dev libxkbcommon-x11-dev libwayland-dev


### 2.6 Fix libEGL, libOpenVG and libGLES links

I've not gone that far in my researches but it seems there's a problem with the linking of these 3 libraries so you rather need to rename them or link them accordingly. In my installation i've decided to rename them.

	sudo cp -PR /opt/vc/lib/libbrcmEGL.so /opt/vc/lib/libEGL.so
	sudo cp -PR /opt/vc/lib/libbrcmGLESv2.so /opt/vc/lib/libGLESv2.so
	sudo cp -PR /opt/vc/lib/libbrcmOpenVG.so /opt/vc/lib/libOpenVG.so


### 2.7 Create a directory for the Qt install
This is where the built Qt sources will be deployed to on the Rasberry Pi. Run the following to create the directory:

	sudo mkdir /usr/local/qt5pi
	sudo chown -R pi:pi /usr/local/qt5pi


## Step 3: Configure your PC
This guide assumes that you have Ubuntu 20.04 already installed on your machine, either natively or running within a virtual machine.

### 3.1 Update your PC
Run the following to update your system and install some needed dependancies:

	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get install gcc git bison python gperf pkg-config gdb-multiarch
	sudo apt-get install build-essential

### 3.2 Set up SSH keys to speed up connecting with the Raspberry Pi

To ease the access on your RPi, i suggest you to make a pair of ssh keys to avoid inserting your password everytime we need to access the machine.

Start a terminal on your RPi and execute the following commands:

	cd /home/pi
	mkdir .ssh
	touch .ssh/authorized_keys
	cd .ssh
	ssh-keygen
	cat /home/pi/.ssh/id_rsa.pub >> /home/pi/.ssh/authorized_keys

At this point you just need to copy the private key on your host machine and access the RPi trough ssh command by passing the key with "-i" option. E.g:

	ssh 192.168.1.1 -l pi -i ~/.ssh/rpi_rsa

Further details about this topic can be found here:
https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md

### 3.3 Set up the directory structure
Create a directory called "rpi" to use as my workspace for the cross-compiler on Ubuntu. Use these commands to create the directory structure:

	sudo mkdir ~/rpi
	sudo mkdir ~/rpi/build
	sudo mkdir ~/rpi/tools
	sudo mkdir ~/rpi/sysroot
	sudo mkdir ~/rpi/sysroot/usr
	sudo mkdir ~/rpi/sysroot/opt
	sudo chown -R 1000:1000 ~/rpi
	cd ~/rpi

The second to last line makes the first user of the computer (hopefully you) the owner of that folder. You can replace the `1000` with your user name if you want to be sure.


## Step 4: Build Preparation & Build

### 4.1 Download Qt sources
Now we can download the source files for Qt. As mentioned before, this guide is for Qt 5.14.2, which is the latest version available at the time of running. Move into tools folder and start downloading the package.

	cd ~/rpi/tools
	sudo wget http://download.qt.io/archive/qt/5.14/5.14.2/single/qt-everywhere-src-5.14.2.tar.xz

Extract the downloaded tar file with the following command:

	sudo tar -xzvf qt-everywhere-src-5.14.2.tar.xz

We need to slightly modify the a mkspec file within the source files to allow us to use our cross compiler. We will copy an existing directory within the source files, and modify the name of the directory and the
contents of the qmake.conf file within that directory to follow the name of our compiler.  

To do this, run the following two command:

	cp -R qt-everywhere-src-5.14.2/qtbase/mkspecs/linux-arm-gnueabi-g++ qt-everywhere-src-5.14.2/qtbase/mkspecs/linux-arm-gnueabihf-g++

	sed -i -e 's/arm-linux-gnueabi-/arm-linux-gnueabihf-/g' qt-everywhere-src-5.14.2/qtbase/mkspecs/linux-arm-gnueabihf-g++/qmake.conf

### 4.2 Download the cross-compiler

Download our toolchain using the following link:

https://app.box.com/s/f8uksyvam238boo8dnguyin547e9l1gl

Once it's downloaded, it's important that you extract it on this specific path '/opt/rpi/' as this toolchain is position dependent.

	cd /opt
	sudo mkdir rpi
	sudo chown -R 1000:1000 rpi
	cd /opt/rpi
	tar -xzvf rpi-gcc-8.3.0_linux.tar.xz

Let's move back into the rpi folder as needed for the next sections:

	cd ~/rpi

### 4.3 Sync our sysroot
We now need to sync up our sysroot folder with the system files from the Raspberry Pi. The sysroot folder will then have all the necessary files to run a system, and therefore also compile for that system.
rsync will let us sync any changes we make on the Raspberry Pi with our PC and vice versa as well. It is convenient because if you make changes to your RPi later and want to update the sysroot folder on your host machine, rsync will only copy over the changes, potentially saving a lot of time.

For now, we want to sync (i.e. copy) over all the relevant files from the RPi. To do this, enter the following commands [change *192.168.1.1* with the IP address for your Raspberry Pi]:

	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/lib sysroot
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/include sysroot/usr
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/lib sysroot/usr
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/opt/vc sysroot/opt

Note: Double check after each of the above commands that all the files have been copied. There will be an information message if there were any issues.
You can run any of those lines again as much as you want if you want to check that all files have been copied. rsync only copies files if any changes have been made.

The `--rsync-path="sudo rsync"` option allows us to access files on the target system (RPi) that may require elevated rights. The rsync configuration changes we made on the RPi itself is what allowed us to use this flag.  

The `--delete` option will delete any files from our host system if they have also been deleted on the RPi. You can probably omit this but I used it as I was troubleshooting library install issues on the RPi.

### 4.4 Fix symbolic links
The files we copied in the previous step still have symbolic links pointing to the file system on the Raspberry Pi. We need to alter this so that they become relative links from the new sysroot directory on the host machine.
We can do this with a downloadable python script. To download it, enter the following:

	cd cd ~/rpi/tools
	wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py

Once it is downloaded, you just need to make it executable and run it, using the following commands:

	sudo chmod +x sysroot-relativelinks.py
	./sysroot-relativelinks.py ../sysroot

### 4.5 Configure Qt Build
Now most of the work we need to set things up has been completed. We can now configure our Qt build.

Let's move into the build directory that we created earlier inside the rpi folder:

	cd ~/rpi/build

Now we need to run the configure script to configure our build. This configure script is actually located inside the qt sources directory. We don't want to build within that source directory as it can get messy, so we will access it from
within this build directory. This is the command you need to run to configure the build, including all the necessary options:

	../tools/qt-everywhere-src-5.14.2/configure -release -opengl es2  -eglfs -device linux-rasp-pi-g++ -device-option CROSS_COMPILE=/opt/rpi/rpi-gcc-8.3.0/bin/arm-linux-gnueabihf- -sysroot ~/rpi/sysroot -prefix /usr/local/qt5pi -extprefix ~/rpi/qt5pi -opensource -confirm-license  -skip qtvirtualkeyboard -skip qtscxml -skip qtspeech -skip qtpurchasing -skip qtgamepad -skip qt3d -skip qtscript -skip qtwayland -skip qtwebengine -skip qtwebview -skip qtwebchannel -nomake tests -make libs -pkg-config -no-use-gold-linker -v -recheck

The configure script may take a few minutes to complete. Once it is completed you should get a summary of what has been configured. Make sure the following options appear:

You may get a warning about QDoc not being compiled. This can be safely ignored unless you specifically need this.

If you have any issues, before running configure again, delete the current contents with the following command (save a copy of config.log first if you need to):

	rm -rf *

The configuration command I provided above skips QtWebEngine as well as other modules which should be unnecessary for our purposes. If you need any of those modules, you can try compile them right now by removing the skip option.
If all looks good and all libraries you need have been installed we can continue to the next section.

### 4.7 Build Qt
Our build has been configured now, and it is time to actually build the source files. Ensure you are still in the build directory, and run the following command:

	make -j 4

The -j 4 option indicates that the job should be spread into 4 threads and run in parallel. I allocated 4 CPU cores to my Ubuntu virtual machine, so I would think the system will make use of that and distribute the workload among the 4 cores.

This process will take some time

Once it is completed, we can install the built package using the following command:

	make install

This should install the files in the correct directories

### 4.8 Deploy Qt to our Raspberry Pi
We can now deploy Qt to our RPi. We will again make use of the rsync command. First move back into the rpi folder using the following command:

	cd ~/rpi

You should now see a new folder named "qt5pi" here. Copy this to the raspberry pi using the following command [replace 192.168.1.1 with the IP address for your Raspberry Pi]:

	rsync -avz --rsync-path="sudo rsync" qt5pi pi@192.168.1.1:/usr/local

N.B: It's important to point out that many of the binaries built during this process and found under /usr/local/qt5pi/bin can't be directly executed on your RPi as they are built to run on your host machine. These are:

- lconvert
- lprodump
- lrelease
- lupdate
- moc
- qdbuscpp2xml
- qdbusxml2cpp
- qlalr
- qmake
- qmlcachegen
- qmlimportscanner
- qmllint
- qmlmin
- qtattributionsscanner
- qvkgen
- rcc
- repc
- uic
- syncqt.pl

### 4.9 Update linker on Raspberry Pi

Enter the following command to update the device letting the linker to find the new Qt library files:

	echo /usr/local/qt5pi/lib | sudo tee /etc/ld.so.conf.d/qt5pi.conf
	sudo ldconfig

The Qt wiki for installing on a Raspberry Pi 2 suggests the following:

	If you're facing issues with running the example,
	try to use 00-qt5pi.conf instead of qt5pi.conf, to introduce proper order.

Something to try if you're having issues running your projects.

That should be it! You have now (hopefully) succesfully installed Qt 5.14.2 on the Raspberry Pi 3B+.

In the next step, we will build OpenCV.


## Step 5: Build OpenCV

It's now time to switch from your host machine to your Raspberry to install OpenCV there. I suggest you to access the desktop of your Raspberry Pi trough HDMI connection or VNC as it will ease the build process.

### 5.1 Install the required development packages

Run the following commands in your Raspberry Pi terminal to install the required packages:

	sudo apt-get install build-essential cmake pkg-config
	sudo apt-get install libjpeg-dev libtiff5-dev libjasper-dev libpng12-dev
	sudo apt-get install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
	sudo apt-get install libxvidcore-dev libx264-dev
	sudo apt-get install libgtk2.0-dev libgtk-3-dev
	sudo apt-get install libatlas-base-dev gfortran

	sudo apt-get install cmake cmake-gui


### 5.2 Download OpenCV source

Run the following commands to download OpenCV 4.4.0 source files in /opt folder:

	cd /opt
	sudo mkdir opencv
	mkdir opencv/build
	sudo chown -R pi:pi opencv
	cd opencv
	wget https://github.com/opencv/opencv/archive/4.4.0.zip

Extract the downloaded zip file with the following command:

	unzip 4.4.0.zip

### 5.3 Build OpenCV from source

Move into build folder and run the following command:

	cd /opt/opencv/build
	cmake-gui ../opencv-4.4.0

Press Configure. Once it has finished, edit the following properties as follows:

	CMAKE_BUILD_TYPE -> Release
	WITH_OPENGL -> True
	WITH_QT -> False
	ENABLE_NEON -> True
	ENABLE_VFP3 -> True  
	ENABLE_NONFREE -> True

Press Configure again. Once it has finished, edit the following properties as follows:

	CPU_BASELINE_DISABLE -> "" (leave it empty)

Configure once again and then press Generate.

It's not time to increase the swap size of your RPi to 2GB otherwise it's most likely that it will hang during build process. Run a terminal on your RPi and execute the following commands:

	sudo /etc/init.d/dphys-swapfile stop
	sudo sed -i -e 's/CONF_SWAPSIZE=100/CONF_SWAPSIZE=2048/g' /etc/dphys-swapfile
	sudo /etc/init.d/dphys-swapfile start

Finally compile OpenCV

	cd /opt/opencv/build
	make -j 4
	sudo make install

### 5.4 Update linker on Raspberry Pi

Enter the following command to update the device letting the linker to find the new OpenCV library files:

echo /usr/local/lib | sudo tee /etc/ld.so.conf.d/opencv.conf
sudo ldconfig

### 5.4 Rsync openCV sources

Go back to your host machine and Rsync new headers and libraries just built:

	cd ~/rpi
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/local/lib sysroot/local
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/local/include sysroot/local
	

## Step 6: Build RaspiCam

It's now time to switch from your host machine to your Raspberry to install RaspiCam there. 

### 6.1 Download RaspiCam source

Run the following commands to download RaspiCam source files in /opt folder:

	cd /opt
	sudo mkdir raspicam
	mkdir raspicam/build
	sudo chown -R pi:pi raspicam
	cd raspicam

	https://sourceforge.net/projects/raspicam/files/latest/download

Extract the downloaded zip file with the following command:

	tar -xzvf raspicam-0.1.9.tar.gz

### 6.3 Build RaspiCam from source

Move into build folder and run the following command:

	cd /opt/rapsicam/build
	cmake ../raspicam-0.1.9

At this point you'll see something like

	CREATE OPENCV MODULE=1
	CMAKE_INSTALL_PREFIX=/usr/local
	REQUIRED_LIBRARIES=/opt/vc/lib/libmmal_core.so;/opt/vc/lib/libmmal_util.so;/opt/vc/lib/libmmal.so
	Change a value with: cmake -D<Variable>=<Value>

	Configuring done
	Generating done
	Build files have been written to: /home/pi/raspicam/trunk/build

If OpenCV development files are installed in your system, then you'll see

	CREATE OPENCV MODULE=1

otherwise this option will be 0 and the opencv module of the library will not be compiled. Needless to say that this option must be 1.

Finally compile the lib:

	make
	sudo make install

### 6.4 Update linker on Raspberry Pi

Enter the following command to update the device letting the linker to find the new OpenCV library files:

	echo /usr/local/lib | sudo tee /etc/ld.so.conf.d/raspicam.conf
	sudo ldconfig

### 6.5 Rsync RaspiCam sources

Go back to your host machine and Rsync new headers and libraries just built:

	cd ~/rpi
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/local/lib sysroot/local
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.1:/usr/local/include sysroot/local

### 6.6 Enable RPi Camera

To enable the RPi camera, after having enabled the interface form raspi-config tool, run the following command on your RPi terminal:

	echo 'bcm2835-v4l2' | sudo tee -a  /etc/modules


## Step 7: Configuring Qt Creator

To configure Qt Creator to be able to build and deploy your projects to your RaspberryPi, follow and copy the following configuration: (note that my qt project is under the path "/home/manfredi/qt_workspaces/myproject")

![Setup GCC](/images/1.png?raw=true "Setup GCC")
![Setup G++](/images/2.png?raw=true "Setup G++")
![Setup Qt version](/images/3.png?raw=true "Setup Qt version")
![Setup RPi Generic Linux device](/images/4.png?raw=true "Setup RPi Generic Linux device")
![Setup Kit](/images/5.png?raw=true "Setup Kit")

Finally setup your .pro file as follows to sucessfully include all the libraries we have installed:


	QT       += core gui

	greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

	CONFIG += c++11

	# The following define makes your compiler emit warnings if you use
	# any Qt feature that has been marked deprecated (the exact warnings
	# depend on your compiler). Please consult the documentation of the
	# deprecated API in order to know how to port your code away from it.
	DEFINES += QT_DEPRECATED_WARNINGS

	# You can also make your code fail to compile if it uses deprecated APIs.
	# In order to do so, uncomment the following line.
	# You can also select to disable deprecated APIs only up to a certain version of Qt.
	#DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0

	SOURCES += \
	    main.cpp \
	    mainwindow.cpp

	HEADERS += \
	    mainwindow.h

	FORMS += \
	    mainwindow.ui

	# Default rules for deployment.
	qnx: target.path = /home/pi/$${TARGET}
	else: unix:!android: target.path = /home/pi/$${TARGET}
	!isEmpty(target.path): INSTALLS += target

	unix:!macx: LIBS += -L$$PWD/../../piQt/sysroot/usr/lib/ -lwiringPi

	INCLUDEPATH += $$PWD/../../piQt/sysroot/usr/include
	DEPENDPATH += $$PWD/../../piQt/sysroot/usr/include

	unix:!macx: LIBS += -L$$PWD/../../piQt/sysroot/usr/local/lib/ -lopencv_core -lopencv_imgcodecs -lopencv_imgproc -lopencv_highgui -lopencv_photo -lopencv_videoio

	INCLUDEPATH += $$PWD/../../piQt/sysroot/usr/local/include/opencv4
	DEPENDPATH += $$PWD/../../piQt/sysroot/usr/local/include/opencv4

	unix:!macx: LIBS += -L$$PWD/../../piQt/sysroot/usr/local/lib/ -lraspicam -lraspicam_cv

	INCLUDEPATH += $$PWD/../../piQt/sysroot/usr/local/include
	DEPENDPATH += $$PWD/../../piQt/sysroot/usr/local/include

## Final touches
After having spent all this time to properly configure you RasberryPi, it's now time to start developing some cool applications with it. That said, you would notice that you won't be able to "stream" the application you're deploying, on your host machine. This is due to some missing configurations we are going to fix now.  

On your RaspberryPi run the following command :

	sudo nano /etc/ssh/sshd_config
	
end enable X11 forwarding by adding the following line to the configuration:

	X11Forwarding yes
	
Save and close the file and restart sshd service:

	sudo systemctl restart  ssh 


On your host machine open a terminal and run the following command:

	ssh -X 192.168.1.61 -l pi -i .ssh/rpi_rsa
	
You'll need to keep this terminal open to maintain the X11 session alive and be able to stream your applications on your host machine.

Move to QtCreator, go to Projects settings -> Build & Run -> Run settings and add the following parameter to command line arguments:

	-platform xcb
	
Then add an environment variable as follows:

	DISPLAY localhost:10.0
	
That's it. Deploy your application and watch the MainWindow directly streamed from your RaspberryPi to your host machine.
