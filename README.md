# Very Particular Docker Mastodon Setup

This repo contains instructions with boilerplate for setting up a single-user Mastodon instance which:

* Runs from container (nginx too)
* Uses official pre-built images, no further building required
* Uses cloud object storage with a distributed cache for serving media files
* Does not send mail


## Host OS

If you don't need to set up a server for running Docker, this step can be skipped.

Start up a VM (VPS) of the latest stable Debian, on your hosting provider of choice.

Follow Mastodon's instructions for initially securing a new server.
  - https://docs.joinmastodon.org/admin/prerequisites/

Follow Docker's instructions for installing from a Debian repository.
  - https://docs.docker.com/engine/install/debian/#install-using-the-repository

After installing Docker, install Docker Compose.

  apt-get install docker-compose

Create a service user in the "sudo" and "docker" groups

  adduser mastodon
  usermod -G sudo,docker mastodon


## Object storage and frontend cache.

If you aren't going to use cloud object storage (S3), this step can be skipped.

Set up a new object storage bucket with your cloud provider of choice. Generate and make note of a shared key and secret for API access.

Set up a distributed frontend cache, such as CloudFront, in front of the bucket. This will be where media files are served from.


## DNS zone

If you aren't going to be running under a custom domain name, this step can be skipped.

Register a new domain name with your registrar of choice, and set up a new zone with your hosting provider.

Create A and AAAA records for the Docker host. Each record type is required (for the letsencrypt challenge). This will be the name of your Mastodon instance. *The name may NOT be changed later*, so make sure you really like it!

If desired, add a subdomain for the distributed frontend cache (e.g. files.example.org) and any other services (mail?) you'll be running.

Verify the zone is resolving correctly, globally, before continuing. A good tool for this is https://dnschecker.org/


## Git setup

Install git, and clone this very repo.

  su - mastodon
  sudo apt-get git
  git clone *{this very repo url}*
  cd *{this very repo}*


## Bootstrap nginx and SSL

Initially based on https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71

  cd provisioning

Edit init-letsencrypt.sh. Insert your domain and other information, and run the script.

  ./init-letsencrypt.sh

Shut down the temporary Compose instances.

  docker-compose down
  cd ..


## Mastodon itself

Use the official Mastodon Docker image to generate a config. Run Mastodon setup, and answer the interactive prompts.

  docker-compose run --rm web bundle exec rake mastodon:setup

Answer "Yes" when prompted to set up a single-user instance. When prompted for SMTP server settings, specify localhost for the mail server, and skip testing. These instructions do not provide for email capability.

Paste the new config into .env.production, and start up the whole cluster in daemon mode.

  docker-compose up -d

If the stars aligned, you should have a running Mastodon instance with SSL support!

Helpful operational and troubleshooting commands:

To view cluster status: docker-compose ps
To tail service logs (e.g. web): docker-compose logs -f web
To shut everything down: docker-compose down
