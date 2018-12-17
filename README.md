## Build live stream server with hls + nginx + ubuntu 16.04
```
#install library
sudo apt-get update		
sudo apt-get install unzip
sudo apt-get install -y build-essential libpcre3 libpcre3-dev libssl-dev

#install nginx with rtmp
mkdir ~/working
cd ~/working
wget http://nginx.org/download/nginx-1.11.6.tar.gz
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
tar -zxvf nginx-1.11.6.tar.gz   
unzip master.zip   
cd nginx-1.11.6    
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-master 
make 
sudo make install 
sudo wget https://raw.github.com/JasonGiedymin/nginx-init-ubuntu/master/nginx -O /etc/init.d/nginx
sudo chmod +x /etc/init.d/nginx
sudo update-rc.d nginx defaults

# install library
sudo apt-get update
sudo apt-get -y install autoconf automake build-essential libass-dev libfreetype6-dev \
libsdl1.2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev \
libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev
sudo apt-get update
sudo apt-get install -y yasm
sudo apt-get install -y  libx264-dev
sudo apt-get install -y  libfdk-aac-dev
sudo apt-get install -y  libvo-aacenc-dev
sudo apt-get install -y  libmp3lame-dev 
sudo apt-get install -y  libopus-dev

#installing Libx265 Library
mkdir ~/ffmpeg_sources
sudo apt-get install -y  cmake mercurial 
cd ~/ffmpeg_sources 
hg clone https://bitbucket.org/multicoreware/x265 
cd ~/ffmpeg_sources/x265/build/linux
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source 
make 
make install 
make distclean

#install Libvpx
cd ~/ffmpeg_sources 
wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.6.0.tar.bz2 
tar xjvf libvpx-1.6.0.tar.bz2 
cd libvpx-1.6.0 
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --disable-examples --disable-unit-tests 
PATH="$HOME/bin:$PATH" make 
make install 
make clean 

#install ffmpeg
sudo add-apt-repository ppa:jonathonf/ffmpeg-4
sudo apt-get update
sudo apt-get install -y  ffmpeg

#config nginx remember change ip address
sudo nano /usr/local/nginx/conf/nginx.conf 
-----------------------------------------nginx.conf----------------------------------------------------------
user root;
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include mime.types;
    default_type application/octet-stream;
	sendfile on;
	server {
        listen 80;
        server_name 192.168.100.131;
		location /hls {
			# Serve HLS fragments
			types {
				application/vnd.apple.mpegurl m3u8;
				video/mp2t ts;                
			}
			root /tmp/;
			add_header Cache-Control no-cache;
			add_header 'Access-Control-Allow-Origin' '*';
		}
		location / {
				root html;
				index index.html index.htm;
			}
		error_page   500 502 503 504  /50x.html;
		location = /50x.html {
			root   html;
		}
	}	
}

rtmp {
	server {
		listen 1935;
		application live {
			live on;
			exec ffmpeg -i rtmp://192.168.100.131/$app/$name -vcodec libx264 -vprofile baseline -x264opts keyint=40 -acodec aac -strict -2 -f flv rtmp://192.168.100.131/hls/$name;
		}
		application hls {
			live on;
			hls on;
			hls_path /tmp/hls/;
			hls_fragment 1s; 
			hls_playlist_length 1s; 
		}
	}
}
------------------------------------------------------------------------------------------------------------

sudo service nginx restart
```

## client.html
```
<!DOCTYPE html>
<html>
<head>
	<title></title>
</head>
<body>
	<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
	<video id="video" muted="muted"></video>
	<script>
		var video = document.getElementById('video');
		var config = {
			liveSyncDurationCount: 1
		};
		var hls = new Hls(config);
		hls.loadSource('http://192.168.100.131/hls/test.m3u8');
		hls.attachMedia(video);
		hls.on(Hls.Events.MANIFEST_PARSED,function() {
		  video.play();
		});
	</script>
</body>
</html>
```
