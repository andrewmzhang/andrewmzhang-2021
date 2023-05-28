---
layout: post
title: Reverse Proxies With Custom ACME
tags: AndyWebServices Tailscale network SSL
category: tech
description: Caveats and solutions regarding setting up a custom ACME server with various reverse proxies
giscus_comments: true
author: Andrew M. Zhang
---
# Intro
This page assumes that you have some custom ACME server (see previous post) and you want a reverse proxy (eg Nginx, HaProxy) to use it to generate certs automatically. 

# SWAG - [docker-swag official](https://github.com/linuxserver/docker-swag)
#### Additional Links:
1. Dockerhub image: [andywebservices/swag](https://hub.docker.com/r/andywebservices/swag)

## Explanation
At time of writing, docker-swag main branch does not support custom ACME servers. Fortunately for the dear reader, I have graciously implemented this feature [andrewmzhang/docker-swag](http://https://github.com/andrewmzhang/docker-swag). It's probably easier to see how the feature works by looking at the [diff](https://github.com/linuxserver/docker-swag/pull/371). This feature introduces 2 new environment variables. One sets the ACME server and the other sets the CABUNDLE. The CABUNDLE is required so that docker trusts the certificate authority without having to install the certificate to the OS truststore, although I still recommend installing the certificate to your OS truststore. 

Docker-swag is a complete solution. It'll renew the certificates once a day and refresh Nginx service to pick up the new certs.

# HAProxy + ACME.sh - [haproxy](https://github.com/haproxy/haproxy)
## Issues
HAProxy suffers several issues. 
1. It cannot provision its own SSL certs, ie it cannot do the ACME dance
2. It hogs port 80 and 443, in order for 3rd party acme to work correctly the acme program needs 80 and 443.
3. It demands that the SSL privkey be in the same file as the cert bundle
4. It cannot tell if the SSL cert has changed on disk, thus users need to send commands to get HAProxy to refresh the certs

To fix part 1, we use `acme.sh`. 
```
# Runs the acme.sh program on port 8888. 
"/home/pi/.acme.sh"/acme.sh --cron --home "/home/pi/.acme.sh" --force --httpport 8888
```
To fix part 2, we need to tell HAProxy to redirect AMCE dance over http to redirect to `acme.sh`

```
"/home/pi/.acme.sh"/acme.sh --cron --home "/home/pi/.acme.sh" --force --httpport 8888
```
frontend public
        bind :::80 v4v6

        # Redirects AMCE challenges towards our other ACME program
	      acl letsencrypt-acl path_beg /.well-known/acme-challenge/
        use_backend letsencrypt-backend if letsencrypt-acl
       
			  # Set the SSL certificate 
        bind :::443 v4v6 ssl crt /home/pi/.acme.sh/octoprint.aws.pem
        option forwardfor except 127.0.0.1
        http-request redirect scheme https code 301 unless { ssl_fc }
        use_backend webcam if { path_beg /webcam/ }
        use_backend webcam_hls if { path_beg /hls/ }
        use_backend webcam_hls if { path_beg /jpeg/ }
        default_backend octoprint
         
        # Sets the amce backend to the 8888 port 
backend letsencrypt-backend
       server letsencrypt 127.0.0.1:8888

```

To fix part 3, concatenate the key and the crt together after running `acme.sh` 
```bash
# This is the code that runs for my Octoprint rpi. 
cat /home/pi/.acme.sh/octoprint.aws.key /home/pi/.acme.sh/octoprint.aws.crt > /home/pi/.acme.sh/octoprint.aws.pem
```

To fix part 4, we need to send some commands to HAProxy to set a new SSL cert. 

```
#!/bin/bash

echo “========================== SET SSL CERT ==========================“
echo "$(cat /home/pi/.acme.sh/octoprint.aws.pem)"
echo -e "set ssl cert /home/pi/.acme.sh/octoprint.aws.pem <<\n$(cat /home/pi/.acme.sh/octoprint.aws.pem)\n" | socat tcp-connect:localhost:9999 -

echo “========================== SHOW SSL CERT - before ==========================“
echo "show ssl cert */home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -

echo “========================== COMMIT SSL CERT ==========================“
echo "commit ssl cert /home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -

echo “========================== SHOW SSL CERT - after ==========================“
echo "show ssl cert /home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -
```
