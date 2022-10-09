# My Very Particular Minimally Viable Docker Mastodon Setup

Stop. Look at the date on this `README`, if there is one. If it's more than a year old, reconsider the common wisdom of continuing.

This repo contains instructions with boilerplate for setting up a single-user Mastodon instance which:

* Runs under `docker-compose`, including `nginx`. Only provisioning steps run on the host.
* Uses official pre-built images, no further building required.
* Uses cloud object storage with a distributed cache for serving media files.
* Does not send mail.


## Host OS

Start up a VM (VPS) of the latest stable Debian, on your hosting provider of choice.

Secure the new host's ports, and install Docker.

Follow Mastodon's instructions for initially securing a new server:
  - https://docs.joinmastodon.org/admin/prerequisites/

Follow Docker's instructions for installing from a Debian repository.
  - https://docs.docker.com/engine/install/debian/#install-using-the-repository

After installing Docker, install Docker Compose.

```sh
apt-get install docker-compose
```

Create a service user in the `sudo` and `docker` groups

```sh
adduser mastodon
usermod -G sudo,docker mastodon
```

## Object storage and frontend cache.

Set up a new object storage bucket (S3) with your cloud provider of choice. Generate and make note of a shared key and secret for API access.

Set up a distributed frontend cache, such as CloudFront, in front of the bucket. This will be where media files are served from.


## DNS zone

Register a new domain name with your registrar of choice, and set up a new zone with your hosting provider.

Create *A* and *AAAA* records for the Docker host. Each record type is required (for the *letsencrypt*( challenge). This will be the name of your Mastodon instance. *The name may NOT be changed later*, so make sure you really like it!

If desired, add a subdomain for the distributed frontend cache (e.g. `files.example.com`) and any other services (mail?) you'll be running.

Verify the zone is resolving correctly, globally, before continuing. A good tool for this is https://dnschecker.org/


## Git setup

Install `git`, and clone `mastodon-boilerplate` (this very repo) into a directory named after your site (replace `example.com`):

```sh
su - mastodon
sudo apt-get git
git clone https://github.com/aayars/mastodon-boilerplate.git example.com
cd example.com
```


## Bootstrap nginx and SSL

Initially based on:
  https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71S

This step starts temporary `nginx` and `certbot` instances, with which we shall try to get a signed SSL certificate issued by *letsencrypt*.

*A* and *AAAA* records must be live and correct at this stage, in order to pass *letsencrypt* challenges.

Make sure you're in the `provisioning` directory for these steps.

```sh
cd provisioning
```

Edit `init-letsencrypt.sh`. Insert the domain name of your Mastodon instance (replace `example.com`) and an admin's email address, and run the script.

```sh
./init-letsencrypt.sh
```

Shut down the temporary `nginx` and `certbot` instances.

```sh
docker-compose down
cd ..
```


## Mastodon &amp; Friends

These steps assume you are back in the top-level directory of the `mastodon-boilerplate` repo.

Just as in the previous step (but with an additional section for the SSL settings), edit `nginx/nginx.conf` and replace all instances `example.com` with your own site's domain name.

Use the official Mastodon Docker image to generate a config. Run Mastodon setup, and answer the interactive prompts.

```sh
docker-compose run --rm web bundle exec rake mastodon:setup
```

Answer "Yes" when prompted to set up a single-user instance. When prompted for SMTP server settings, go with the default of `localhost` for the mail server, and skip testing. These instructions do not provide for email capability, but you can set that up if you really want to.

The interactive script will print out the contents of a new config to your console. Paste the new config into `.env.production`, and start up the whole cluster in daemon mode.

```sh
docker-compose up -d
```

If the stars aligned, you should have a running Mastodon instance with SSL support!

Helpful operational and troubleshooting commands (while in the top-level directory of the `mastodon-boilerplate` repo):

To view cluster status: `docker-compose ps`.

```sh
$ docker-compose ps
         Name                        Command                  State                        Ports
------------------------------------------------------------------------------------------------------------------
exampleorg_certbot_1     /bin/sh -c trap exit TERM; ...   Up             443/tcp, 80/tcp
exampleorg_db_1          docker-entrypoint.sh postgres    Up (healthy)
exampleorg_nginx_1       /docker-entrypoint.sh /bin ...   Up             0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
exampleorg_redis_1       docker-entrypoint.sh redis ...   Up (healthy)
exampleorg_sidekiq_1     /usr/bin/tini -- bundle ex ...   Up (healthy)   3000/tcp, 4000/tcp
exampleorg_streaming_1   /usr/bin/tini -- node ./st ...   Up (healthy)   3000/tcp, 4000/tcp
exampleorg_web_1         /usr/bin/tini -- bash -c r ...   Up (healthy)   3000/tcp, 4000/tcp
```

To tail service logs for a named service (e.g. sidekiq, streaming, web): `docker-compose logs -f web`

To shut down everything: `docker-compose down`
