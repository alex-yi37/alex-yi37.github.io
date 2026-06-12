---
title: Setting up a single-node Docker swarm project
description:
date: 2026-02-12
tags:
  - docker
  - react
draft: true
---

I'm currently running a dockerized [Inertia Rails](https://inertia-rails.dev/) application on a $6/month deployed with a tool called `kamal`. It can integrate with a container registry (in my case Dockerhub) and deploy from your local machine to a configured VPS (virtual private server, just some VM - virtual machine - provided by a hosting company). I believe the rough flow is: run `kamal deploy` => this builds the app image locally => pushes the built image to a configured image repository => the vps pulls the built image down => the tool starts the built app image as a container, as well as a `kamal-proxy` container that handles setting up TLS, doing "zero downtime" deploys, etc. From a [previous post](/blog/setting-up-nginx-for-docker) <= meta: trying to link to another blogpost on my site per [this 11ty issue, no idea if it works](https://github.com/11ty/eleventy/issues/1207#issuecomment-761606802) =>, I installed nginx on the Docker host to receive web traffic, then pass the traffic to the `kamal-proxy` container, which then passes the request to the running app container because I want to run multiple apps on the VPS, so this nginx step will also come up in this setup.

# Motivation

- some guy in a devops/sre Discord server I'm in is a big advocate of using Docker Swarm for hosting side projects he deploys. He even has a Terraform setup with Docker and a repo for it. I've tried reading what the repo, but I don't know enough about Terraform or how the setup itself works to really understand it at the moment. He seems pretty knowledgeable and I think it seems like a nicer alternative to Docker Compose (another option I tried looking into but couldn't figure out how to provide secure env vars without integrating with another platform)
- Docker Swarm can store secrets (my definition being any environment variables I don't want other people to read) so I don't have to figure out how to get sensitive data into my application. The only way I know how to provide env vars to Docker before learning about this was storing them in a `.env` file and I couldn't figure out how to get that data into the Docker container "securely". If the vars are hardcoded in the image, then they can [apparently be read if someone has access to the image](https://www.reddit.com/r/docker/comments/1bpjni5/comment/kwwa615/), and if I have to copy an `.env` file to my Docker host, that [can supposedly be read as well](https://www.reddit.com/r/docker/comments/1bpjni5/comment/kww9ia5/) (I would assume depending on linux user permissions and such, I really don't have a good understanding of the this myself). This is all to say, I will take these claims at face value at the moment because my goal is to get this site set up and simply accept that Docker Swarm offers a more secure way to provide secure env vars than the methods I mentioned previously.

# Steps

1. created a Tanstack start repo locally using the cli: `npm create @tanstack/start@latest` and pushed it to Github
2. created an A record in DNS registrar (porkbun in this case) that points the subdomain `snacc-tracc.sunasuna.cafe` to the ip of the VPS. At the moment, I have porkbun pointing to AWS Route53 for its nameservers so I will have to change it back so porkbun handles DNS for the root domain, and then believe I can set a NS record on porkbun to point a subdomain I'm using (`alex-resume.sunasuna.cafe` for cloud resume challenge hosted on AWS) to a Route53 hosted zone. We'll see if that actually works how I want it to.

- subtask: update my AWS infra that I'm using for my cloud resume challenge to account for moving nameservers back to porkbun. I think I'll realistically only have to change things that are pointing to the current hosted zone (all `sunasuna.cafe`) to the new hosted zone (only `alex-resume.sunasuna.cafe`). Probably have to update Cloudfront and ACM, and that's it?
  - create `alex-resume.sunasuna.cafe` hosted zone
  - copy Cloudfront alias A record from `sunasuna.cafe` hosted zone to new hosted zone
  - request new ACM cert that is validated against `alex-resume` subdomain
  - disable Cloudfront distro
  - remove association between Cloudfront distro and old ACM cert, link CF to new ACM cert
  - delete old ACM cert
  - add new cert to new hosted zone as CNAME record - the UI in the ACM console didn't let me select the hosted zone to add records, so I went in Route53 and added the record and value provided in the ACM console for the new cert
  - in old hosted zone, added NS record pointing `alex-resume` subdomain to new hosted zone nameservers. I realize I didn't include a `.` at the end of each nameserver domain when I did this (looked at the original NS record that came with each hosted zone and noticed a `"."` at the end of each one), tried adding it to the nameserver domains in the NS record in the old hosted zone, but for some reason they don't get saved. Maybe not necessary in this case?
  - tried hitting `alex-resume.sunasuna.cafe` but got Chrome (not AWS) nxdomain error page. Might have to wait a bit for DNS propagation? Try again in like an hour - [this SO comment says NS transfer may take from 1 to 24 hours (is an answer from 2017 though)](https://stackoverflow.com/a/43799692). Been more than an hour (started keeping track around 11:30 PM on 2/11/2026), come back to this later

```bash

# the columns Route53 provides in the console view
Record Name | Type | Routing Policy | Differentiator | Alias | Value/Route traffic to | TTL
# relevant record to copy over from old zone to new zone
# custom domain for cloudfront
alex-resume.sunasuna.cafe | A | Simple | - | Yes (is an alias record) | some_hash.cloudfront.net. | -

# ACM cert domain validation record that I don't need to copy, but want to include here for reference
_another_long_hash.alex-resume.sunasuna.cafe | CNAME | Simple | - | No | _long_hash_1.other_hash_2.acm-validations.aws. | 300

# copy over from old zone to porkbun dns settings
# CNAME to point blog subdomain to github pages site hosted with 11ty site generator
alex-blog.sunasuna.cafe | CNAME | Simple | - | No | alex-yi37.github.io | 300

```

- might have to provision new ACM cert that points to new hosted zone?
- update ACM cert in Cloudfront distro to utilize new cert?
- create NS record in original hosted zone that points `alex-resume` to nameservers for new hosted and see if can reach the site
  - this step has not been working. created NS record pointing to new hosted zone nameservers, then added cloudfront alias record and got nxdomain. Deleted alias records in new zone, then tried adding an A record pointing to DO vps ip and still got nxdomain. Deleted cloudfront alias in new zone, then deleted `alex-resume` NS record in old zone. Added A record in old zone pointing `alex-resume` to DO vps ip and that actually started getting routed after waiting for a little bit.
  - eventually, deleted the NS record in old hosted zone that was pointing to new hosted zone nameservers then remade it, and then `alex-resume` domain was reachable again! Why...?
- make porkbun primary DNS for domain again
- add a NS record in porkbun pointing to nameservers provided by new AWS hosted zone, as well as `alex-blog` CNAME from old hosted zone
- delete old hosted zone on AWS

3. Figuring out Dockerfile. 2/15/2026 - going to try copying `react-router` [default template](https://github.com/remix-run/react-router-templates/blob/961b011535d8243c233407210c7c1f4bf2ea9266/default/Dockerfile) and adapt that.

```Dockerfile
# react-router default Dockerfile template
# believe dev dependencies required to build the app, probably bc typescript is a devDependency
FROM node:20-alpine AS development-dependencies-env
COPY . /app
WORKDIR /app
RUN npm ci

FROM node:20-alpine AS production-dependencies-env
COPY ./package.json package-lock.json /app/
WORKDIR /app
RUN npm ci --omit=dev

FROM node:20-alpine AS build-env
COPY . /app/
COPY --from=development-dependencies-env /app/node_modules /app/node_modules
WORKDIR /app
RUN npm run build

FROM node:20-alpine
COPY ./package.json package-lock.json /app/
COPY --from=production-dependencies-env /app/node_modules /app/node_modules
COPY --from=build-env /app/build /app/build
WORKDIR /app
CMD ["npm", "run", "start"]
```

I took the above Dockerfile and replaced the `COPY --from=build-env /app/build /app/build` line and changed it to read as `COPY --from=build-env /app/.ouput /app/.ouput` as the `.output` directory is where built files are output by default with the Tanstack Start template I used. I was able to run `docker build -t test .` to build the image and call it `test`, then run it with `docker run -p 3000:3000`, and finally access the site locally at `http://localhost:3000` so I assume the build works correctly. It appears that the output server code can serve the built static files (js, css, etc) as-is, so I didn't try doing anything else to serve static files differently. Since this is an app I don't expect to get any real traffic, this feels alright but could try using a CDN in the future or try having nginx on the host vps serve the static files if I really need to.

4. create and push a production image of the app to a container repository service on merges to `main` branch, probably have to write a Github Action for this, and create a `compose.yml` file to use for deploying a swarm stack. Since I'm only building an image for my web app and using sqlite, I will only have one service, but I would like to have docker swarm create and track the volume used to back the sqlite database.

- created a private Dockerhub repository for `snacc-tracc`
- created a docker context to be able to (hopefully) run commands that can access my `compose.yml` from my local machine: `docker context create digital-ocean-vps --docker "host=ssh://gomi@my-vps-ip"` and it ran successfully. Ran `docker --context digital-ocean-vps ps` on my local machine and it listed `kamal-proxy` and the `gomi-guesser-web` containers, so worked as expected
  - otherwise, I'm not really sure of a good way to have docker reference `compose.yml` without copying it directly to the docker host
- built and tagged my production image by running `docker build -t alex7543/snacc-tracc .`, then pushed to DockerHub by running `docker push alex7543/snacc-tracc` (the `:latest` tag is implied when no explicit tag added I believe).
  - 2/20/2026 - got error: `denied: requested access to the resource is denied`
    - ran `docker login` and it said my login succeeded with existing credentials, not which credenitals those are. From running `docker-credential-osxkeychain list` from [this docker forum post](https://forums.docker.com/t/hello-can-you-please-help-me-the-cli-command-to-find-out-the-docker-user-login-info-in-the-powershell/140100/3), it out put `{"https://index.docker.io/v1/":"gomigroup"}` so I assume that means I signed into the gomi-group account automatically. Ran `docker login -u alex7543` and entered a password when prompted to login to the `snacc-tracc` DockerHub user I plan to use. Ran `docker-credential-osxkeychain list` again and it output `"alex7543"` so seems like it worked
    - ran `docker push alex7543/snacc-tracc` again and the image got successfully pushed, checked that it was up on DockerHub web
- ran `docker --context digital-ocean-vps swarm init --advertise-addr my-vps-ip-address`. Tried without the `--advertise-addr` at first but got an error since apparently there are multiple addresses on interface eth0. Did not save the join taken produced by the command since I plan to use a single node setup
- added a `docker-stack.yml` file to be used by docker swarm stack, then from my local machine ran `docker --context digital-ocean-vps stack deploy --with-registry-auth --compose-file docker-stack.yml snacc-tracc`
  - seemed to run properly after I fixed a syntax error in `docker-stack.yml`. In a `volume` mapping entry, I had a space between the volume name and the path in the container. I removed the space and the deploy command ran to completion
- ssh'd into the vps and ran `docker ps` and `docker image ls` but doesn't seem like the container was up or that an image was pulled. Ran `docker service ls` and I saw that the `snacc-tracc` service was created but there were 0/1 replicas up
- ran `docker stack ps snacc-tracc --no-trunc` on the vps and got output, part of which said `"No such image: alex7543/snacc-tracc:latest"`
- added `--with-registry-auth` flag as part of the `docker stack deploy command` and output saying the service was updated

2/21/2026

- wanted to try re-deploying with a fresh stack, so ran `docker stack rm snacc-tracc` on the vps, and then removed `snacc-tracc` images that had been pulled down
- from local machine, ran `docker --context digital-ocean-vps stack deploy --with-registry-auth --compose-file docker-stack.yml snacc-tracc` again

```yaml
# legacy version used from docs example https://docs.docker.com/guides/swarm-deploy/#describe-apps-using-stack-files
version: "3.7"
services:
  web:
    image: alex7543/snacc-tracc
    ports:
      - "3000:3000"
    volumes:
      # try to bind
      - snacc-tracc: /app/storage/production.sqlite
volumes:
  snacc-tracc:
```

# Note to self - 2/18/2026

- in app code, need to define a file path for sqlite db file. In the generated starter code for dev, that file path is provided as an environment variable: `process.env.DB_URL` that is asserted to be a non-null value using the TypeScript `!` operator - `process.env.DB_URL!` - and provided as an environment variable in a `.env` file as `dev.db`. For a value of `dev.db`, I believe that will create a sqlite file at the root of the project directory called `dev.db` (honestly not sure). I want to map a volume to `/storage/production.db` when the app is actually deployed, so I think I need to change the file path to something like `./storage/$process.env.DB_URL`, so does that mean I should supply the string `production.db` as an environment variable in my docker swarm stack file?

Delete below later

```bash
# DockerHub instructions to push a tag? Different from image?
# To push a new tag to this repository:
docker push alex7543/snacc-tracc:tagname
```

TODO:

- GH action to run tests
- GH action to build image and push it to GH image registry
- GH action to ssh into vps and pull GH image from registry (maybe leave for later, can hopefully do it manually?)
  - ssh into vps
  - run docker command to pull image
  - run docker swarm command to

5. TODO: created `/etc/nginx/conf.d/snacc-tracc.sunasuna.cafe.conf` file and included basic `proxy_pass localhost:port_number;` directive pointing to the port that I configure the app to listen on, then ran `sudo systemctl reload nginx` to load the config. At this point, nginx should be sending traffic to an app that doesn't exist, but only over http. If I try to access `snacc-tracc.sunasuna.cafe` in a browser, I get an error saying the site isn't secure because it is only communicating over http. To start serving traffic over https, I need to acquire a TLS cert using Letsencrypt and Certbot

```bash
# /etc/nginx/conf.d/snacc-tracc.sunasuna.cafe.conf config
server {
  server_name snacc-tracc.sunasuna.cafe;

  location / {
    proxy_pass localhost:9090;
  }
}
```

6. TODO: run Certbot command to set up automatic TLS cert issuing and renewal for `snacc-tracc.sunasuna.cafe` coupled with nginx - `sudo certbot --nginx -d snacc-tracc.sunasuna.cafe`. This automatically updates the config file written in the previous step and also sets up a cron job that will try to renew the cert some time before it is actually set to expire. At this point, I should be able to visit my domain and get a 502 response back from nginx because the app I'm proxying traffic to doesn't exist yet
7. write a

# To figure out

- all the other stuff to actually get Docker Swarm running
- secrets with Docker Swarm - https://docs.docker.com/engine/swarm/secrets/
- how to ensure prod sqlite file is persisted in a docker volume. Also how to run migrations against prod sqlite file
- I'm not sure I want to deploy on every push or merge to `main` branch. Can I create a `prod` branch that deploys the app whenever there are pushes or merges into it? And in practice, I would only make PRs against prod from `main`
