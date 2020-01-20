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
./configure <paste-arguments-copied-before> --add-dynamic-module=../naxsi-$NAXSI_VERSION/naxsi_src/
make modules
```

From above command, a file `ngx_http_naxsi_module.so` will be created under `/tmp/nginx-$NGINX_VERSION/objs` directory. Copy this file into `/etc/nginx/modules`. Copy `/tmp/naxsi-$NAXSI_VERSION/naxsi_config/naxsi_core.rules` into `/etc/nginx`, this file can be found under `naxsi-$NAXSI_VERSION/naxsi_config`.

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

Naxsi works on a per-location basis, meaning you can only enable it inside a location under each vhost. Create another file to store basic rules or [`checkrules`](https://github.com/nbs-system/naxsi/wiki/checkrules-bnf) to enable `naxsi` either in learning mode or production mode as well as whitelist rules. Create file `/etc/nginx/naxsi.rules` and put following content:

```
SecRulesEnabled; #enable naxsi
LearningMode; #<----- learning mode, turn off to start blocking unintended request
LibInjectionSql; #enable libinjection support for SQLI
LibInjectionXss; #enable libinjection support for XSS

# Checkrules
# CheckRule must be present at location level.
# Check https://github.com/nbs-system/naxsi/wiki/checkrules-bnf for more details
DeniedUrl "/RequestDenied"; #the location where naxsi will redirect the request when it is blocked
CheckRule "$SQL >= 8" BLOCK; #the action to take when the $SQL score is superior or equal to 8
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 5" BLOCK;
CheckRule "$UPLOAD >= 5" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
```

Open server block or vhost file to include naxsi rules. May refer following example:

```
server {
...

    location / {

        include '/etc/nginx/naxsi.rules';
	
	# naxsi best if use with reverse proxy
	# pass traffic to apache port 8080
	proxy_pass http://127.0.0.1:8080; 
        ....
    }

    
    location /RequestDenied { # this is the location where naxsi redirect if block the request
        internal;
        return 403; // Or return any file.
    }

    ...

}
```

When request to the server initiated by web browser, the request will handle by `naxsi`, if something unintended happened, `naxsi` will redirect the request into `/RequestDenied`, turns out showing 403 or anything. Restart nginx to reflect changes:

```
# check for config file
nginx -t

# Reload nginx service
service nginx reload
```

Test basic XSS script:

```
curl 'http://<your_server_ip>/?q="><script>alert(0)</script>'
```

Verify that its actually block the request by reading `/var/log/nginx/error.log`

```
2019/09/06 03:05:05 [error] 21356#0: *1 NAXSI_FMT: ip=<your_server_ip>&server=<your_server_ip>&uri=/&learning=0&vers=0.56&total_processed=1&total_blocked=1&block=1&cscore0=$SQL&score0=8&cscore1=$XSS&score1=8&zone0=ARGS&id0=1001&var_name0=q, client: <your_server_ip>, server: localhost, request: "GET /?q="><script>alert(0)</script> HTTP/1.1", host: "<your_server_ip>"
```

Or try with basic SQL injection:

```
curl 'http://<your_server_ip>/?q=1" or "1"="1"'
```

And observe the `error.log`:

```
2019/09/06 03:07:05 [error] 21356#0: *2 NAXSI_FMT: ip=<your_server_ip>&server=<your_server_ip>&uri=/&learning=0&vers=0.56&total_processed=2&total_blocked=2&block=1&cscore0=$SQL&score0=40&cscore1=$XSS&score1=40&zone0=ARGS&id0=1001&var_name0=q, client: <your_server_ip>, server: localhost, request: "GET /?q=1" or "1"="1" HTTP/1.1", host: "<your_server_ip>"
```

If you found something like above inside `error.log` file, it is proved that you've configure `naxsi` properly. While in learning mode, `naxsi` only log the error without actually block the request. To enable in production site, just uncomment the option `# LearningMode;`.

### Creating Whitelist

Not only that, `naxsi` even provide an option to only block the certain request. By default `naxsi` will block any suspicious request but if you want `naxsi` blocking only for certain request, thats `whitelist` features comes to the rescue. 

To do that, we can use external plugin called `nx_util` which is made available by using python code, download [here](https://code.google.com/archive/p/naxsi/downloads). Or just head over to this documention for [whitelist](https://github.com/nbs-system/naxsi/wiki/whitelists-bnf) for more details. Execute following command:

```
cd /tmp
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/naxsi/nx_util-1.1.tgz
tar -zxf nx_util-1.1.tgz
cd nx_util-1.1.tgz
```

And following command to get the constructed rules:

```python
python nx_util.py -c nx_util.conf -l /var/log/nginx/* -o
```

After run above command, it will create an output and copy everything:

```
########### Optimized Rules Suggestion ##################
# total_count:4 (40.0%), peer_count:1 (100.0%) | ; in stuff
BasicRule wl:1008 "mz:$URL:/|$ARGS_VAR:q";
# total_count:2 (60.0%), peer_count:1 (100.0%) | parenthesis, probable sql/xss
BasicRule wl:1010 "mz:$URL:/|$ARGS_VAR:q";
```

Paste following content at the end of `/etc/nginx/naxsi.rules` and turns of `#LearningMode`. This time, `naxsi` will only blocks the request if the request matched with the rules defined above. Check [rules](https://github.com/nbs-system/naxsi/wiki/rules-bnf) for more details. Once everything in place, reload nginx

```
service nginx reload
```

Done.



