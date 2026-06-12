---
title: Trying to install nginx on a VPS to host multiple Docker apps
description:
date: 2026-02-04
tags:
  - nginx
  - linux
  - docker
draft: true
---

I have a Ruby on Rails app currently hosted at `sunasuna.cafe` running on a cheap Digital Ocean droplet (it is a VPS aka virtual private server) that has been up for a few months at this point.

Lead up to this. Followed [an article on Digital Ocean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04) to create a `sudo` user and disable logging in as the `root` user, and enabled a firewall to allow ssh and http/https traffic. Then followed part of (this guide)[https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04] to install `nginx`.

2/4/2026 - installed nginx, had to shutdown Docker container running `kamal-proxy` since it was listening on port 80 and nginx failed to start because the port was in use. Currently, can hit my droplet's ip in a browser to see the default nginx page, but trying to visit `sunasuna.cafe` gives an error page in the browser. Possibly because of how nginx is configured by default?

2/5/2026

- apparently nginx looks for the `Host` header to determine how to route a request based on `server_name` directive in config, but that header is only mandatory in HTTP/1.x requests. According to [this thread](https://superuser.com/questions/1659248/how-does-browser-know-which-version-of-http-it-should-use-when-sending-a-request), browsers only support HTTP/2 on TLS connections. In Chrome's case, when it establishes a TLS connection it sends a list of supported protocols, then the server it's talking to responds with the protocol it wants to use
- HTTP/2 can send requests without the `Host` header, instead using the `:authority` pseudo-header per [this thread](https://superuser.com/questions/1687956/why-i-use-chrome-request-a-site-url-do-not-see-host-header)
- in [this article](https://www.linode.com/docs/guides/how-to-configure-http-2-on-nginx/), it states that to use HTTP/2 with nginx, you have to enable https with something like `Certbot`

2/6/2026

- in nginx, can create config in `conf.d` directory in file named something like `mysite.conf`, or create a file in `sites-available` and then create a symlink to the file in `sites-enabled`. I opted for a file in conf.d called `conf.d/sunasuna.cafe.conf`. I basically copied in config from default file that came with nginx install, then in Chrome had to finagle it so I could request my site over http instead of https, but was able to see a file get served.

Main things

- install nginx
- configure a server block with `server_name my_domain.com www.my_domain.com;`, then have it `proxy_pass` to `kamal-proxy` over `localhost`
- try to configure `kamal-proxy` to not use tls, and change the port mapping to match the `proxy_pass` in nginx config, e.g. if we `proxy_pass http:localhost:8080` then map port `8080` on the docker host running `kamal-proxy` to port `80` in `kamal-proxy` container
- question: other than doing https for us without writing any config, what does running `kamal-proxy` container give us? I believe can do things like zero downtime deploy, but if I don't care about that does it make sense to not run `kamal-proxy` at all and just pass traffic from nginx directly to the container running my `rails` app?
- question: still don't entirely understand how changing the `https_port` that `kamal-proxy` was mapped to on docker host caused my `kamal deploy` to work. Seemed as if the `rails` app itself wasn't responding to the `/up` endpoint to verify healthchecks, so I assumed the `rails` container was down or something
