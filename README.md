**Building an LTE hostspot / video streaming device using an Raspberry pi a USB ZTE modem with a T-mobile sim card and a Reo Link IP camera.**

My goal for this project was to create a camera streaming device with a wifi hotspot to use at our local RC track to provide internet connection for the control computer as well as provide a camera feed to upload to the livetime scoring system.

**Hardware I used:**

Raspberry Pi 3b
ZTE LTE USB Modem
Sandisk 64G SD card
ReoLink bullet IP camera
Ethernet cable
Power supply for both the camera and pi

**What I am trying to accomplish here:**

My goal is to let the pi provide a DHCP addresses to devices over both the ethernet port and wifi so that devices can connect to the internet through both.

**Setup:**

To start, you will need to flash your SD card with the latest version of raspbian. Once this is complete you can go to the next step.


I personaly plan to access this system over SSH, and over the internet. A piece of software you can use for this is provided by https://remote.it, this will install on the system and show up in the console which you can then use to connect to the device over SSH. This is very handy as the ZTE modem does not allow port forwarding.

Once the pi is setup and ready to go you can login and start getting the system configured. 

Start by installing the bridge utils and configuring a bridge on the system.

```
sudo apt install bridge-utils
```

from here you will want to edit the /etc/network/interfaces file, make sure you put your own settings / ip addresses here.
```
# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

# automatically connect the wired interface
auto eth0
allow-hotplug eth0
iface eth0 inet manual

# automatically connect to the lte modem on usb0
auto usb0
allow-hotplug usb0
iface usb0 inet dhcp

# automatically connect the wireless interface, but disable it for now
auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wireless-power off

# create a bridge with both wired and wireless interfaces
auto br0
iface br0 inet static
        address 10.3.141.1
        netmask 255.255.255.0
        bridge_ports eth0 wlan0
        bridge_fd 0
        bridge_stp off
```        
Next you will want to configure the dhcp server, this one only requires a few lines.
```
sudo apt install dnsmasq
```
```
# which network interface to use
interface=br0

# which dhcp IP-range to use for dynamic IP-adresses
dhcp-range=10.3.141.100,10.3.141.150,12h
```

After this you will want to configure the access point for devices to connect to your system, this will be done with hostapd.

```
sudo apt install hostapd
systemctl unmask hostapd
systemctl enable hostapd
```

**edit the hostapd file at /etc/hostapd/hostapd.conf**

```
bridge=br0

interface=wlan0
driver=nl80211
ssid=(YOURSSID)

hw_mode=g
channel=7
ieee80211d=0
country_code=US

ieee80211n=1
ieee80211ac=1
wmm_enabled=1

wpa=2
auth_algs=1
wpa_key_mgmt=WPA-PSK
wpa_passphrase=(YOUR PASSWORD)
rsn_pairwise=CCMP
```

you will need to then modify the ```/etc/default/hostapd``` ffile and add the following line.

```DAEMON_CONF="/etc/hostapd/hostapd.conf"```


next you will want to setup iptables to allow forwarding of traffic.

you will want to uncomment the line...  
```net.ipv4.ip_forward=1``` 
in the /etc/sysctl.conf


**Next, run the following commands, the ttl-set command is important for mobile hotspot, if you do not have this the data will be consumed as hotspot data and not out of your unlimited data.**

```
sudo apt install iptables iptables-persistent
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -j ACCEPT
sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65
sudo iptables-save > /etc/iptables/rules.v4
sudo iptables-persistent save
sudo reboot
```

Once this is complete. you should be able to give a DHCP address to your camera. You will want to run the command ```arp -a``` to look for anything that has an address on your network, one of these should be the camera. (if you cannot find the camera, connect with a windows machine to the hotspot and search for the camera this way)

with the camera connected (and you have logged in and set a password) you should be able to now connect and stream this camera. One thing i have found helpful is to use screen.

 ```apt-get install screen```

once installed you can run screen to back ground the camera streaming process so you can close the pi ssh session with out the video stream stopping.

run 
```screen```

once here you will want to run something similar, i have tested this with livetime scoring as well as twitch. Youtube should also work. You will want to adjust your settings to fit your needs but i have found this configuration works well.

```
ffmpeg -rtsp_transport udp -i rtsp://admin:123456@10.3.141.10:554//h264Preview_01_main -framerate 15 -video_size 1920x1080 -vcodec libx264 -maxrate 1984k -bufsize 10M -g 50 -codec:v copy -c:a aac -b:a 128k -f flv rtmp://live-phx.twitch.tv/app/streamkey
```

to get out of the screen session you will want to hit ```cntl + a``` ```cntrl + d``` this will disconnect you from the session, to re connect you can type ```screen -r SCREENSESSIONNUMBER``` which the screen session number is gathered by running ```screen -list```



**Sources**

```
https://snikt.net/blog/2019/06/22/building-an-lte-access-point-with-a-raspberry-pi/
https://github.com/niski84/t-mobile-throttle-defeat
https://support.reolink.com/hc/en-us/articles/360007010473-How-to-Live-View-Reolink-Cameras-via-VLC-Media-Player
https://www.raspberrypi.org/forums/viewtopic.php?t=207639
```
