# Introduction
Collection of hardening methods or steps can be used to secure our web applications from attackers.

## Web Server
- Nginx
- Apache

## Nginx
For the nginx, we'll be using `naxsi` to secure our web application. `naxsi` will seat in front of our web application. Whenever request comes in from client side(browser, ajax, curl, etc), all these request will be handled by `naxsi` to inspect whether the request(cookies, url, etc) is containing malicious code or anything's bad before forwarding request to the nginx. If for some reasons, those requests contains any bad attemps, `naxsi` will redirect those request to somewhere else.
