Original tutorial credit: Steve Kuekes

# Jasper on Raspian Buster Software Installation Guide
This install process was tested on a Raspberry Pi Model 3 B+ and Model 4 using the Raspian Buster Lite ISO (https://downloads.raspberrypi.org/raspbian_lite_latest). This tutorial used a USB microphone and speakers that plug into the audio jack and uses pocketsphinx for the speech to text and festival for text to speech. This method is fairly secure in that there is no traffic sent to 3rd parties.


# Burn image onto SD Card
Boot the Raspian image on the RPi device and sign in.
Use `raspi-config` to enable the network adapter or wireless as needed and enable SSH to make thigns easier.
Plug in the microphone and speakers then restart the Pi.

    $ ssh pi@<ip-address> # replace this address with the address of your Pi

# Update and install some utilities
    $ sudo apt-get -y update
    $ sudo apt-get -y upgrade
    $ sudo apt-get install -y git-core python-dev bison libasound2-dev libportaudio-dev python-pyaudio
    $ sudo apt-get install -y python-setuptools
    $ sudo python /usr/lib/python2.7/dist-packages/easy_install.py pip

# Create/edit ALSA configuration file:
    $ sudo touch /lib/modprobe.d/jasper.conf &&  sudo chown pi /lib/modprobe.d/jasper.conf && sudo cat > /lib/modprobe.d/jasper.conf<<EOF
    #Loads USB audio before the internal soundcard
    options snd_usb_audio index=0
    options snd_bcm2835
    #Makes sure the sound cards are ordered the correct way in ALSA
    options snd slots=snd_usb_audio,snd_bcm2835
    EOF
    $ sudo shutdown -r now

# Test the microphone and speakers:
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
    LD_LIBRARY_PATH="/usr/local/lib""
    export LD_LIBRARY_PATH
    PATH=$PATH:/usr/local/lib/
    export PATH
    EOF
    $ source .bashrc

### The authoritative repo for jasper is at https://github.com/jasperproject/jasper-client.git but it hasn't been updated since 2017.
### This tutorial uses a forked repo:
    $ git clone https://github.com/aplawson/jasper-client.git jasper
    $ wget https://raw.githubusercontent.com/aplawson/jasperbuster/master/jasper.patch
    $ cd jasper
    $ patch -p1 < ../jasper.patch
    $ cd ..

Install needed Python libraries
    $ sudo pip install --upgrade setuptools
    $ sudo pip install -r jasper/client/requirements.txt        #### this takes a while

Create the Jasper profile
    $ cd ~/jasper/client
    $ python populate.py
    $ cd ~

# Install PocketSphinx and the utilities we will need
    $ sudo apt-get install pocketsphinx python-pocketsphinx pocketsphinx-en-us
    $ sudo apt-get install subversion autoconf libtool automake gfortran g++

# Install CMUSphinx
    $ svn co https://svn.code.sf.net/p/cmusphinx/code/trunk/cmuclmtk/
    $ cd cmuclmtk/
    $ ./autogen.sh
    $ make
    $ sudo make install
    $ cd ~

# Install Phonetisaurus, m2m-aligner and MITLM
## To use the Pocketsphinx STT engine, you also need to install MIT Language Modeling Toolkit, m2m-aligner and Phonetisaurus (and thus OpenFST)
    $ wget http://distfiles.macports.org/openfst/openfst-1.3.4.tar.gz
    $ wget https://github.com/mitlm/mitlm/releases/download/v0.4.1/mitlm_0.4.1.tar.gz
    $ wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/m2m-aligner/m2m-aligner-1.2.tar.gz
    $ wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/phonetisaurus/is2013-conversion.tgz
    $ tar -xvf m2m-aligner-1.2.tar.gz
    $ tar -xvf openfst-1.3.4.tar.gz
    $ tar -xvf is2013-conversion.tgz
    $ tar -xvf mitlm_0.4.1.tar.gz

# Patch and build OpenFST.
    $ wget http://www.kuekes.com/openfst.patch
    $ cd openfst-1.3.4/
    $ patch -p 1 < ../openfst.patch
    $ sudo ./configure --enable-compact-fsts --enable-const-fsts --enable-far --enable-lookahead-fsts --enable-pdt
    $ sudo make install # come back after a really long time *** you will get a bunch of gcc warnings, but it is ok, let them go
    $ cd ~

# Build m2m-aligner
    $ cd m2m-aligner-1.2/
    $ sudo make
    $ cd ~

# Build mitlm
    $ cd mitlm-0.4.1/
    $ sudo ./configure
    $ sudo make install
    $ cd ~

# Build Phonetisaurus
    $ cd is2013-conversion/phonetisaurus/src
    $ sudo make
    $ cd ~

# Move some files around
    $ sudo cp ~/m2m-aligner-1.2/m2m-aligner /usr/local/bin/m2m-aligner
    $ sudo cp ~/is2013-conversion/bin/phonetisaurus-g2p /usr/local/bin/phonetisaurus-g2p

# Download and build the Phonetisaurus FST model
    $ wget https://www.dropbox.com/s/kfht75czdwucni1/g014b2b.tgz
    $ tar -xvf g014b2b.tgz
    $ cd g014b2b/
    $ ./compile-fst.sh
    $ cd ..
    #Rename the folder for convenience
    $ mv ~/g014b2b ~/phonetisaurus

# Install Festival for TTS
    $ sudo apt-get install festival festvox-don

# Final edits the Jasper profile.yml
    $ cd .jasper
    $ vi profile.yml
    
### Update the profile as shown. X's and 1's represent private user-specific information. It could already be set.
    carrier: vtext.com
    first_name: xxxxx
    gmail_address: xxxx@gmail.com
    gmail_password: xxxx
    last_name: xxxxx
    phone_number: '1111111111'
    prefers_email: false
    stt_engine: sphinx
    timezone: America/Los_Angeles
    pocketsphinx:
        hmm_dir: '/usr/share/pocketsphinx/model/en-us/en-us'
        fst_model: '../phonetisaurus/g014b2b.fst'
    tts_engine: festival-tts

# Reboot
    $ sudo shutdown -r now

# Start Jasper
    $ cd jasper
    $ ./jasper.py

# Itsa done
