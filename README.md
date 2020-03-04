# Overview
Jasper is an opensource project designed to make the process of creating a self-standing "Alexis"-style assistant. Because the project hasn't seen any new commits since 2017, it's a pretty tough project to get working for experienced engineers and few have ever successfully completed the task. But the value of a standalone voice assistant that does not interact with 3rd parties is important. Thus the creation of this DIY (do it yourself) guide. 

This guide (with special thanks to Steve Keukes who provided the original draft) steps through the process of building a fully-functional Jasper installation on a Raspberry Pi. The guide should work if the user can be trusted to type commands *as written* with no additional creativity added. Every repository used by this guide has been safely forked and linked within GitHub. All files used have been stored in an accompanying SourceForge project owned by the author of this guide (Adam Peacock) at https://sourceforge.net/projects/jasperclient/files/.

#### Package details are listed [HERE](https://github.com/aplawson/jasperbuster/blob/master/README.md#package-details)

# Hardware Used in this Tutorial (tutorial -reportedly- works on a Model 4 as well)
* Raspberry Pi 3 B+ (https://www.amazon.com/gp/product/B07BDR5PDW)
* CanaKit Raspberry Pi 3 B+ (B Plus) with Premium Clear Case and 2.5A Power Supply (https://www.amazon.com/gp/product/B00MARDJZ4)
* Edimax EW-7811Un 150Mbps 11n Wi-Fi USB Adapter (https://www.amazon.com/gp/product/B003MTTJOY)
* Samsung 32GB 95MB/s (U1) microSDHC EVO Select Memory Card (https://www.amazon.com/gp/product/B06XWN9Q99)
* Conference Microphone, OmniDirectional Dual Capsules, USB Mic (https://www.amazon.com/gp/product/B07LC3FT5B)
* Speaker system (with 3.5mm jack) for playback

# Jasper on Raspian Buster Software Installation Guide
This install process was tested on a Raspberry Pi Model 3 B+ and Model 4 using the Raspian Buster Lite ISO (https://sourceforge.net/projects/jasperclient/files/2020-02-13-raspbian-buster-lite.zip). The file can be downloaded from the public repo but to emnsure the file versions do not change, the IMG file was moved to SourcerForge to ensure the version of the initial OS is known/tested. This tutorial used a USB microphone and speakers that plug into the audio jack and uses PocketSphinx for the speech-to-text (STT) and Festival for text-to-speech (TTS). This method is fairly secure in that there is no traffic sent to 3rd parties.

# Feeling Lazy?
A fully-functional Jasper installation was burned to an IMG file that works on any RPI 3B+ (possibly other models but they haven't been tested). The file can downloaded directly at https://sourceforge.net/projects/jasperclient/files/jasper-rpi.zip/download and written to an SD-card using any image burning software (i.e. Etcher). The username and password were not changed from the Raspbian defaults. (Username: `pi` Password: `raspberry`)

# Burn image onto SD Card
Boot the Raspian image on the RPi device and sign in.
Use `raspi-config` to enable the network adapter or wireless as needed and enable SSH to make thigns easier.
Plug in the microphone and speakers then restart the Pi.

    $ ssh pi@<ip-address>

# Update and install some utilities
    $ sudo apt-get -y update
    $ sudo apt-get -y upgrade
    $ sudo apt-get install -y vim git-core python-dev bison libasound2-dev libportaudio-dev python-pyaudio dos2unix
    $ sudo apt-get install -y python-setuptools
    $ sudo python /usr/lib/python2.7/dist-packages/easy_install.py pip

# Create/edit ALSA configuration file:
    $ sudo touch /lib/modprobe.d/jasper.conf &&  sudo chown pi /lib/modprobe.d/jasper.conf && sudo cat > /lib/modprobe.d/jasper.conf<<EOF
    #Loads USB audio before the internal soundcard
    options snd_usb_audio index=0
    options snd_bcm2835
    #snd_bcm2835 = built-in audio jack
    #snd_usb_audio = USB-connected microphone
    #Makes sure the sound cards are listed as desired in ALSA
    #Verify order (and what ALSA sees) by entering "alsamixer" at the command line
    #If a device is not listed, reboot the device and try again.
    #options snd slots=<capture_device>,<playback_device>
    options snd slots=snd_usb_audio,snd_bcm2835
    EOF
    $ sudo reboot

# Test the microphone and speakers:
Note: This tutorial mentions `alsamixer` as a handy tool to test playback and microphone gain/volume. However, if certain aspects of alsamixer don't work quite right (i.e. F4 Capture), feel free to troubleshoot but don't add software binaries not mentioned in this guide unless you know what you're doing. * Remember this tool is a matter convenience - not a requirement.
* To start alsamixer: `$ alsamixer`
* Press `F6` to select microphone
* Press `F4` to capture
* Press up arrow to increase microphone gain
* To record: `$ arecord test.wav`, speak into the microphone, end recording: `Ctrl + C`
* To playback: `$ aplay -D hw:1,0 test.wav`

#### Edit and source `.bash_profile`:
    $ touch .bash_profile && cat>>.bash_profile<<EOF
    export LD_LIBRARY_PATH="/usr/local/lib"
    EOF
    $ source .bash_profile

#### Edit and source `.bashrc`:
    $ touch .bashrc && cat>>.bashrc<<EOF
    LD_LIBRARY_PATH="/usr/local/lib"
    export LD_LIBRARY_PATH
    PATH=$PATH:/usr/local/lib/
    export PATH
    EOF
    $ source .bashrc

### The authoritative repo for jasper is at https://github.com/jasperproject/jasper-client.git but it hasn't been updated since 2017. This tutorial uses a forked repo:
    $ git clone https://github.com/aplawson/jasper-client.git jasper
    $ wget https://raw.githubusercontent.com/aplawson/jasperbuster/master/jasper.patch
    $ cd jasper && patch -p1 < ../jasper.patch
    $ cd

#### Install needed Python libraries
    $ sudo pip install --upgrade setuptools
    $ sudo pip install -r jasper/client/requirements.txt        #### this takes a while

#### Create the Jasper profile
    $ cd ~/jasper/client
    $ python populate.py
    $ cd

# Install PocketSphinx and some necessary utilities
    $ sudo apt-get install -y pocketsphinx python-pocketsphinx pocketsphinx-en-us
    $ sudo apt-get install -y subversion autoconf libtool automake gfortran g++

# Install CMUSphinx
    $ wget https://master.dl.sourceforge.net/project/jasperclient/cmuclmtk.zip
    $ unzip cmuclmtk.zip && chmod 777 -R cmuclmtk && cd cmuclmtk/ && find . -type f -exec dos2unix -k -s -o {} ';'
    $ ./autogen.sh
    $ make
    $ sudo make install
    $ cd

# Install Phonetisaurus, m2m-aligner and MITLM
## MIT Language Modeling Toolkit, m2m-aligner, Phonetisaurus and OpenFST are needed for the Pocketsphinx STT engine
    $ wget https://master.dl.sourceforge.net/project/jasperclient/openfst-1.3.4.tar.gz
    $ wget https://master.dl.sourceforge.net/project/jasperclient/mitlm_0.4.1.tar.gz
    $ wget https://master.dl.sourceforge.net/project/jasperclient/m2m-aligner-1.2.tar.gz
    $ wget https://master.dl.sourceforge.net/project/jasperclient/is2013-conversion.tgz
    $ tar -xvf m2m-aligner-1.2.tar.gz
    $ tar -xvf openfst-1.3.4.tar.gz
    $ tar -xvf is2013-conversion.tgz
    $ tar -xvf mitlm_0.4.1.tar.gz

# Patch and build OpenFST.
    $ wget https://raw.githubusercontent.com/aplawson/jasperbuster/master/openfst.patch
    $ cd openfst-1.3.4 && patch -p 1 < ../openfst.patch
    $ sudo ./configure --enable-compact-fsts --enable-const-fsts --enable-far --enable-lookahead-fsts --enable-pdt
    #Next step takes very long time. There might be some gcc warnings -- just ignore
    $ sudo make install
    $ cd

# Build m2m-aligner
    $ cd m2m-aligner-1.2/
    $ sudo make
    $ cd

# Build mitlm
    $ cd mitlm-0.4.1/
    $ sudo ./configure
    $ sudo make install
    $ cd

# Build Phonetisaurus
    $ cd is2013-conversion/phonetisaurus/src
    $ sudo make
    $ cd

# Move some files around
    $ sudo cp ~/m2m-aligner-1.2/m2m-aligner /usr/local/bin/m2m-aligner
    $ sudo cp ~/is2013-conversion/bin/phonetisaurus-g2p /usr/local/bin/phonetisaurus-g2p

# Download and build the Phonetisaurus FST model
    $ wget https://master.dl.sourceforge.net/project/jasperclient/g014b2b.tgz
    $ tar -xvf g014b2b.tgz && mv ~/g014b2b ~/phonetisaurus && cd phonetisaurus
    $ ./compile-fst.sh
    $ cd

# Install Festival for TTS
    $ sudo apt-get install -y festival festvox-don

# Final edits the Jasper profile.yml
### The use of Facebook and Gmail credentials are optional if email access or Facebook integration is desired
    $ cat > .jasper/profile.yml<<EOF
    carrier: carriername
    first_name: first-name
    gmail_address: emailaddress
    gmail_password: emailpassword
    last_name: last-name
    phone_number: '1234567890'
    prefers_email: false
    stt_engine: sphinx
    timezone: America/Los_Angeles
    pocketsphinx:
        hmm_dir: '/usr/share/pocketsphinx/model/en-us/en-us'
        fst_model: '../phonetisaurus/g014b2b.fst'
    tts_engine: festival-tts
    EOF

# Reboot
    $ sudo reboot

# Start Jasper
    $ cd jasper
    $ ./jasper.py

#### END OF GUIDE

# PACKAGE DETAILS

Temporarily required for install/compiling process:

`vim` To edit files manually if needed

`git-core` To clone repositories

`dos2unix` To remove non-UNIX characters from some files that break functionality. Not needed after script is cleaned up

`g++` To compile packages from source

`python-setuptools/pip` To install python 2.7 modules

`automake` To assist with make/compile process(es)

`autoconf` To assist with make/compile process(es)

Required for Jasper to function (post-install):

`python-dev` To install python-development tools used by Jasper.
> WARNING: The Jasper project is written in Python 2.7 which is deprecated. The viability of this guide is dependent on the eventual migration from Python 2 to Python 3.
