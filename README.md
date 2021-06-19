# install-rtmp-nginx
## Step 1: Login via SSH to the server
Login as root or a user with sudo access on the server. If you are using a user with sudo access, then add sudo before each command in the tutorial
## Step 2: Download required software
Start by updating the apt repository:   
<pre> apt-get update </pre>
If this is a new server, you should consider updating the system software:
<pre> apt-get Upgrade </pre>
We'll then install the required software:
<pre>apt-get install -y git build-essential ffmpeg libpcre3 libpcre3-dev libssl-dev zlib1g-dev </pre>
## Step 3: Clone Module
<pre>git clone https://github.com/sergey-dryabzhinsky/nginx-rtmp-module.git</pre>
## Step 4: Download Nginx
Copy the latest download link from the Nginx website and decompress the files:
<pre>wget http://nginx.org/download/nginx-1.20.1.tar.gz
tar -xf nginx-1.17.6.tar.gz
cd nginx-1.17.6</pre>
## Step 5: Configure Nginx
<pre>./configure --prefix=/usr/local/nginx --with-http_ssl_module --add-module=../nginx-rtmp-module
make -j 1
make install</pre>
## Step 6: Configure Nginx
<pre>rm /usr/local/nginx/conf/nginx.conf
nano /usr/local/nginx/conf/nginx.conf</pre>
Copy the following contents into the file and save it:
<pre>worker_processes  auto;
events {
worker_connections  1024;
}

# RTMP configuration
rtmp {
server {
listen 1935; # Listen on standard RTMP port
chunk_size 4000;

application show {
live on;
# Turn on HLS
hls on;
hls_path /mnt/hls/;
hls_fragment 3;
hls_playlist_length 60;
# disable consuming the stream from nginx as rtmp
deny play all;
}
}
}
http {
sendfile off;
tcp_nopush on;
directio 512;
default_type application/octet-stream;

server {
listen 8080;

location / {
# Disable cache
add_header 'Cache-Control' 'no-cache';

# CORS setup
add_header 'Access-Control-Allow-Origin' '*' always;
add_header 'Access-Control-Expose-Headers' 'Content-Length';

# allow CORS preflight requests
if ($request_method = 'OPTIONS') {
add_header 'Access-Control-Allow-Origin' '*';
add_header 'Access-Control-Max-Age' 1728000;
add_header 'Content-Type' 'text/plain charset=UTF-8';
add_header 'Content-Length' 0;
return 204;
}

types {
application/dash+xml mpd;
application/vnd.apple.mpegurl m3u8;
video/mp2t ts;
}

root /mnt/;
}
}
}
</pre>
## Step 7: Start Nginx
Check if everything is OK
<pre> /usr/local/nginx/sbin/nginx -t </pre>
Start Nginx
<pre>/usr/local/nginx/sbin/nginx</pre>
## Step 8: Install compiled Nginx as System Service
Go to **/usr/lib/systemd/system/** and create a new service file like this:
<pre>[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
</pre>
Enable the service for future startups
<pre>systemctl enable nginx.service</pre>
Et voila, we get nginx with rtmp

## Step 9: Startt Streaming
This server can stream from a variety of sources including a static file, webcam, etc.
We previously installed ffmpeg. We'll start streaming [4MP@15fpsAVC-10.mp4](https://drive.upm.es/index.php/s/6rTdZcfaeEvoQdt/download "link title") which simulates a 4MP IP camera stream to our http://localhost/show/stream  
<pre>ffmpeg -re -i 4MP@15fpsAVC-10.mp4 -vcodec libx264 -vprofile high -g 30 -acodec aac -strict -2 -f flv rtmp://localhost/show/stream</pre>
If we want to play looped we may modify ffmpeg command as this way
<pre>... -re -stream_loop -1 -i ... <pre>
