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

Download latest(or the same version installed on your server) `nginx` source code from [here](https://nginx.org/download/). At the time of writing, i'll be using `v1.14.0`
```
cd /tmp
wget https://nginx.org/download/nginx-1.14.0.tar.gz
```

Extract tarball
```
tar -xvf nginx-x.x.x.tar.gz
```

Download the lateset `naxsi` source code from [here](https://github.com/nbs-system/naxsi/releases). At the time of writing, i'll be using `v0.56`
```
cd /tmp
wget https://github.com/nbs-system/naxsi/archive/0.56.tar.gz -O naxsi
```

Extract tarball
```
tar -xvf naxsi
```

Once downloaded, head over to `nginx` folder
```
cd /tmp/nginx-1.14.0
```

Export environment variable to ease installation
```
export nginx_version=nginx 
```


