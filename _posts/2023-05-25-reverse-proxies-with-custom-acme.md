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

**EDIT: This section was updated on 2024-04-17. The previous instructions were out of date**

HAProxy suffers several issues.

1. It cannot provision its own SSL certs, ie it cannot do the ACME dance
2. It hogs port 80 and 443, in order for 3rd party acme to work correctly the acme program needs 80 and 443.
3. It demands that the SSL privkey be in the same file as the cert bundle
4. It cannot tell if the SSL cert has changed on disk, thus users need to send commands to get HAProxy to refresh the certs

Fortunately, `acme.sh` has some helpers that make this procedure relatively painless

```bash
# Register an account thumbprint. This will produce a thumbprint. Copy that value
./acme.sh --register-account --server https://step-ca.internal/acme/acme/directory -m myemail@example.com

# Edit the following into /etc/haproxy/haproxy.cfg
global
        [...]
        stats socket /var/run/haproxy/admin.sock level admin mode 660    # This command lets ./acme.sh communicate to HAProxy to reload SSL certs
        setenv ACCOUNT_THUMBPRINT 'THE VALUE COPIED FROM THE PREVIOUS COMMAND'

frontend public
        bind :::80 v4v6
        bind :::443 v4v6 ssl crt /etc/haproxy/certs/ strict-sni  # This allows haproxy to boot without certs, which you wont have initially
        # The directive below means when the certificate authority navigates to my.domain.internal/.well-known/acme-challenge/ HAProxy will reply with the account thumbprint
        http-request return status 200 content-type text/plain lf-string "%[path,field(-1,/)].${ACCOUNT_THUMBPRINT}\n" if { path_beg '/.well-known/acme-challenge/' }

# Do the ACME dance, ACME will write some config files under ~/.acme.sh/mydomain.internal_ecc. Note the deploy-hook and --days 1
./acme.sh --stateless --issue -d my.domain.internal --server https://step-ca.internal/acme/acme/directory --ca-bundle ~/my_root_ca.crt --deploy --deploy-hook haproxy --days 1
# Remember to update cron
./acme.sh --install-cronjob
```

Remember to check the .acme.sh config files, namely the `Le_RenewalDays` value. It defaults to 60 days, but step-ca certs default expires in 1 day, so you'll need to mess with this value

### Resources

1. https://www.haproxy.com/blog/haproxy-and-let-s-encrypt
