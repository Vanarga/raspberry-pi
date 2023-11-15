# Install Foundry VTT on raspberry pi
## Introduction
1. [Foundry Virtual Table Top](https://foundryvtt.com/)
1. [Foundry Virtual Table Top Knowledge Base](https://foundryvtt.com/kb/)
1. [Self-Hosting Foundry VTT on a Raspberry Pi](https://dracoli.ch/posts/foundry-rpi/)
1. [Certbot Instructions](https://certbot.eff.org/lets-encrypt/debianbuster-nginx)
1. [Updating IP Address In OpenDNS With Raspberry Pi](https://nothing.golddave.com/2016/03/04/updating-ip-address-in-opendns-with-raspberry-pi/)
1. [D&D 5eTools: Plutonium/Rivet FoundryTool Install](https://wiki.5e.tools/index.php/)
1. [Using Foundry VTT without a System and PDF Character Sheets](https://www.youtube.com/watch?v=fi8m07QGQ-8)
1. [Updated Foundry Basics Part 1 - Installing, Updating, and Creating Our World](https://www.youtube.com/watch?v=-aHlApa1nUA)
## Foundry VTT Installation
1. **Setup a Samba Share.**
    1. sudo mkdir -m 1777 ~/share
    2. sudo apt-get update
    3. sudo apt install -y samba samba-common-bin
    4. sudo nano /etc/samba/smb.conf
    5. **Paste the code below into the file.**
    ```
    [share]
    Comment = pi shared folder
    Path = /home/pi/share
    Browseable = yes
    Writeable = yes
    only guest = no
    create mask = 0777
    directory mask = 0777
    Public = yes
    ```
    6. Ctrl-X
    7. Press **'Y'** to save.
    8. sudo smbpasswd -a pi
    9. sudo service smbd restart
2. **Install Foundry VTT dependencies.**
    1. sudo apt-get install -y libssl-dev
    2. curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
    3. sudo apt-get install -y nodejs
3. **Create application and user data directories.**
    1. cd $HOME
    2. mkdir foundryvtt
    3. mkdir share/foundrydata
4. **Get the Foundry VTT download link and License key.**
    1. Browse to [Installation Guide](https://foundryvtt.com/article/installation/)
    2. Login
    3. Select your name in the upper right corner of the screen.
    4. Select Purchased Licenses.
    5. Make note of the Purchased Software License code. You will need it later.
    6. Click the little chain icon next to the Node.js installation link to copy the link location.
5. **Download the Foundry VTT Software.**
    1. cd foundryvtt
    2. wget -O foundryvtt.zip "`Paste Foundry Link from step 4f`"
       1. Note: make sure to include the ""
    3. unzip foundryvtt.zip
6. **Set up the Foundry VTT Service.**
    1. sudo nano /etc/systemd/system/foundryvtt.service
    2.  **Paste the code below into the file.**
    ```
    [Unit]
    Description=Foundry VTT
    After=network.target

    [Service]
    ExecStart=/usr/bin/node /home/pi/foundryvtt/resources/app/main.js --dataPath=/home/pi/share/foundrydata
    WorkingDirectory=/home/pi/foundryvtt
    StandardOutput=inherit
    StandardError=inherit
    Restart=always
    User=pi

    [Install]
    WantedBy=multi-user.target
    ```
    1. Ctrl-X
    2. Press **'Y'** to save.
    3. sudo systemctl daemon-reload
    4. sudo service foundryvtt start
    5. sudo systemctl enable foundryvtt
7. **You can now browse to the ip address of your foundry server.**
#### References:
1. [Self-Hosting Foundry VTT on a Raspberry Pi](https://dracoli.ch/posts/foundry-rpi/)
1. [Foundry Knowledge Base](https://foundryvtt.com/kb/)
1. [How to run FoundryVTT on a Raspberry Pi 4 using nginx and foundryvtt-docker](https://github.com/orangetruth/foundryvtt-raspberry-pi/blob/main/README.md)