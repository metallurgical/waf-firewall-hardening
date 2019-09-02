# Introduction
Collection of hardening methods(web application filtering) or steps can be used to secure our web applications from attackers.

## Web Server
- Nginx: ubuntu
- Apache: ubuntu

## Nginx
For the nginx, we'll be using `naxsi` to secure our web application. `naxsi` will seat in front of our web application. Whenever request comes in from client side(browser, ajax, curl, etc), all these request will be handled by `naxsi` to inspect whether the request(cookies, url, etc) is containing malicious code or anything's bad before forwarding request to the nginx. If for some reasons, those requests contains any bad attemps, `naxsi` will redirect those request to somewhere else. Lets get started.

Update system package
```
apt-get update
```

Install required dependencies
```
apt-get install libpcre3-dev libssl-dev unzip build-essential daemon libxml2-dev libxslt1-dev libgd-dev libgeoip-dev
```

Export environment variable to ease of installation. At the time of writing, i'll be using vesion `0.56` and `1.14.0` for `naxsi` and `nginx` respectively
```
export NAXSI_VERSION=0.56
export NGINX_VERSION=1.14.0
```

Download latest(or the same version installed on your server) `nginx` source code from [here](https://nginx.org/download/)
```
cd /tmp
wget https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz
```

Extract tarball
```
tar -xvf nginx-$NGINX_VERSION.tar.gz
```

Download the lateset `naxsi` source code from [here](https://github.com/nbs-system/naxsi/releases).
```
cd /tmp
wget https://github.com/nbs-system/naxsi/archive/$NAXSI_VERSION.tar.gz -O naxsi_$NAXSI_VERSION.tar.gz
```

Extract tarball
```
tar -xvf naxsi_$NAXSI_VERSION.tar.gz
```

Important notes here, `naxsi` must be compile with the same configurations with the `nginx`. To get the existing arguments or config for `nginx`, issue below command and copy the arguments after wording `configure arguments:`

```
nginx -V

# Above command will resulting the output like following
nginx version: nginx/1.14.0
built by clang 10.0.0 (clang-1000.11.45.5)
built with OpenSSL 1.0.2q  20 Nov 2018 (running with OpenSSL 1.0.2s  28 May 2019)
TLS SNI support enabled
configure arguments: --prefix=/usr/local/Cellar/nginx/.......
.....
```

Head over to `nginx` folder, to compile the source
```
cd /tmp/nginx-$NGINX_VERSION
./configure <paste-arguments-copied-before> --add-dynamic-module=../naxsi-$NAXSI_VER/naxsi_src/
make modules
```

From above command, a file `ngx_http_naxsi_module.so` will be created under `objs` directory. Copy this file into `/etc/nginx/modules`. Copy `naxsi_core.rules` into `/etc/nginx`, this file can be found under `naxsi-$NAXSI_VERSION/naxsi_config`.

Load module `ngx_http_naxsi_module.so` inside `/etc/nginx/nginx.conf` as well as `naxsi_core.rules`

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

load_module /etc/nginx/modules/ngx_http_naxsi_module.so; # <------------ load naxsi module

events {
	worker_connections 768;
	# multi_accept on;
}

http {
	client_max_body_size 100M;
  	include /etc/nginx/naxsi_core.rules; # <----------- load naxsi core rules
  
	##
	# Basic Settings
	##
  	.......
  	.......
}
```

Naxsi works on a per-location basis, meaning you can only enable it inside a location under each vhost.



