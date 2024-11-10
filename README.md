# Live stream from raspberry pi camera with overlay from text file

This project create live streaming service with video overlay.
For hw part it uses Raspberry Pi 3 model B v1.2 + IR-Cut Night Vision Camera OV5658 5MP 1080p + temperature sensor.
CCD size: 1/4 inch, Aperture(F): 1.8, Focal Length: 3.6 mm (adjustable), Field of View: 72 degrees.
Camera support: up to 2592 x 1944 pixels, also supports 1080p30, 720p60 and 640x480p60/90 video more info: [camera](https://www.ovt.com/sensors/OV5658).

For case used Raspberry Pi HQ Camera Case from [this](https://learn.adafruit.com/raspberry-pi-hq-camera-case).
For software part used: python+ffmpeg+nginx. For ffmpeg part using codec h264_omx for reduce cpu usage.
To avoid limitations from power supply input, 500W PSU was used as power source, which can supply serious 17A on +5V output. 
Measured voltage was +5.120 V at RPi pins. 
Connection between Pi and PSU was done using short cable with AWG18 wires.

<img src="/images/pi-external-power.jpg" alt="External power" width="640" height="854">

For make your raspberry pi cooler, set up aluminum heat-sink. 
Covered SoC and IO chip
<img src="/images/pi-cooling.jpg" alt="cooling" width="854" height="640">

# Raspberry pi settings

Download OS for raspberry pi from [this](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit) 
Write flash drive: sudo dd if=Raspberry.iso of=/dev/sdb status=progress
Connect raspberry to external monitor throught HDMI
Default login: pi password: raspberry
Connect to WiFi, enable 1-Wire, CSI camera port. Next expand flash size: Advanced Options -> «A1 Expand Filesystem».
Check camera:

```
$ vcgencmd get_camera
supported=1 detected=1
```
Reboot.

# Temperature sensor DS18B20 setup

First, connect to raspberry pi board

![DS18b20 pin out](/images/ds18b20-pinout.png)
![Raspberry pi GPIO schematics](/images/GPIO-pinout.png)
![Connect ds18b20 to raspberry pi GPIO header](/images/pi-connect.jpg)
![More GPIO header](/images/rpiblusleaf.png)

For read data from sensor, first search it on a bus
```
$ ls /sys/bus/w1/devices
28-011936686111  w1_bus_master1
```
for read data:
```
$ cat /sys/bus/w1/devices/28-011936686111/temperature
3250
```
Divide by 1000 for get celcius

Next setup ffmpeg utility

`sudo apt update`
`sudo apt install -y ffmpeg`

And check installed version:

`ffmpeg -version`


# Nginx setup

Use Ubuntu or some ubuntu flawor. First remove apache if it installed:
`sudo apt remove apache2`

And then install nginx from repos. It configured systemd services add sites-enabled and etc.
Later it will be used for updated version nginx buildede from sources.

`sudo apt install nginx`

Backup nginx and build new from source for support streaming operation
`sudo mv /usr/sbin/nginx /usr/sbin/nginx.old`
Check nginx setup:
`sudo nginx -t`
Console output:
```
pi@raspberrypi:~ $ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Start systemd service:
`sudo service nginx start`
Check nginx startup page:
```
pi@raspberrypi:~ $ curl -v 127.0.0.1
* Expire in 0 ms for 6 (transfer 0x1f41960)
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x1f41960)
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1
> User-Agent: curl/7.64.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.14.2
< Date: Sun, 10 Nov 2024 03:34:54 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Sun, 10 Nov 2024 03:33:16 GMT
< Connection: keep-alive
< ETag: "6730297c-264"
< Accept-Ranges: bytes
```
Stop for update:
`sudo service nginx stop`

# Build Nginx from source for support rtmp module

Install support packages:
`sudo apt-get install libpcre3 libpcre3-dev libssl-dev`

# Get nginx source
`wget http://nginx.org/download/nginx-1.17.8.tar.gz`

extract it to nginx folder:
`tar -xvf nginx-1.17.8.tar.gz nginx`

# Download rtmp-module
`wget https://github.com/arut/nginx-rtmp-module/zipball/master -O nginx-rtmp-module-master.zip`
extract:
`unzip nginx-rtmp-module-master.zip -d nginx-rtmp-module-master`


# Go to nginx sources, run this:

`cd nginx`

And then run:

```
./configure --prefix=/usr --add-module=../nginx-rtmp-module-master/arut-nginx-rtmp-module-xxx/ \
--pid-path=/var/run/nginx.pid --conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log \
--with-http_ssl_module
```
Note: nginx-rtmp-module-master contain subfolder arut-nginx-rtmp-module-xxx, rename to actual version in configure script.

# make it
```
make
sudo make install
sudo cp ../nginx-rtmp-module-master/arut-nginx-rtmp-module-xxx/stat.xsl /etc/nginx/
```
# Run it
Check nginx config before start:
`sudo nginx -t`
If no erros start nginx systemd service
`sudo service nginx start`

Edit config, add streaming section name - streamer, enable HLS.
The stream accessed thru /tmp

`sudo vim /etc/nginx/nginx.conf`

```javascript
user www-data;
worker_processes auto;
pid /run/nginx.pid;

rtmp_auto_push on;

rtmp {
    record all;
    live on;
    server {
        listen 1935;
        application streamer {
            hls on;
            hls_path /tmp/hls;
            hls_fragment 5s;
        }
    }
}
```

# Enable access to /tmp for browser
Edit `/etc/nginx/sites-enabled/default` and add 
```
 server {

 listen 80;
    listen [::]:80 ipv6only=on default_server;
    server_name localhost;
    index index.html index.htm index.php;

    location /hls {
        root /tmp;
    }
```

# ffmpeg prepare

The broadcast prepare using the ffmpeg utility.

* take live video from the camera
* overlay the sound from the mp3 file by looping it (broadcasting without an audio track usually doesn't work)
* apply captions from the sensors.txt file to the image, 
* update sensors.txt from another process
* upload the received data to the server, which can be Youtube or our remote ubuntu server with nginx installed.

script for local stream: /home/pi/username/streamLocal.sh
for youtube streaming: /home/pi/username/streamYoutube.sh

# Utility scripts
camera.py used for read data from temperature sensor DS18B20 and write it to sensor.txt file

# Another script
`raspivid -o - -t 0 -w 800 -h 600 -fps 12  | cvlc -vvv stream:///dev/stdin --sout '#rtp{sdp=rtsp://:5000/}' :demux=h264`
