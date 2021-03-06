# Feedbin in Docker
## Info About This Fork

This is a fork of [angristan/feedbin-docker](https://github.com/angristan/feedbin-docker) with a couple minor changes and fixes, as well as clearer documentaiion. I am in no way an "expert" but I'm using this as a learning oppurtunity. I hope this helps anyone trying to selfhost Feedbin!

Please note, this setup is rather involved. While I am working to make this as easy as possible, this may not be the best choice if you do not at least have some experience with Docker and selfhosting. 

Also feedbin is pretty resource hungry, so you will not have a good time trying to host this on an entrylevel VPS.

That being said, contributions are always welcome and I do think this is one of the better (selfhostable) feedreaders out there.

----
Self-host [Feedbin](https://github.com/feedbin/feedbin) with Docker. Feedbin is a web based RSS reader. It's an open-source Ruby on Rails software.

Feedbin's main goal is not to be easily self-hostable, and it was quite hard getting all of the services to work. During the process of creating `feedbin-docker`, I made [a few contributions to the upstream project](https://github.com/feedbin/feedbin/commits?author=angristan) to make it self-hostable. Other have taken other approaches by forking it, but all the projects I found on GitHub were abandonned and weren't working anymore.

I chose to run it in Docker because of all the services required to run Feedbin.

Here is a breakdown of all the containers:

* `web`: the puma rails app
* `workers`: some of the sidekiq workers for background processing
* `refresher`: sidekiq worker for refreshing feeds
* `image`: sidekiq worker to find thumbnails
* `extract`: nodejs service to extract article content from full web pages
* `camo`: a node reverse proxy to prevent mixed content
* `minio`: object storage for images, favicons, imports
* `redis`: cache, store sidekiq queues and stats
* `memcached`: cache
* `postgresql`: database
* `elasticsearch`: full text search
* `caddy`: https-enabled reverse proxy

As you can see it's a lot. Technically, you can give up on a few of them without breaking Feedbin:

* `image`: you won't have thumbnails, which is not that important depending on your appearance settings.
* `camo`: your browser will make the requests directly to the websites. Less privacy and risk or mixed content.
* `elasticsearch`: you won't have full text search

You can also replace `caddy` with another reverse proxy, but caddy is really handy.

## Setup

### 0. Prerequisites

* I recommend a server with **more than 2 GB of RAM**. Otherwise you will likely have OOM kills.
* You will also need a domain setup. In your DNS, create A records pointing to your IP address for `feedbin`, `camo.feedbin`, `minio.feedbin`, and `extract.feedbin`. It is important to you do this **before** attempting installation. Otherwise, this will not work.
* Clone this repo, by running:
```sh
git clone https://github.com/gotmax/feedbin-docker.git
```

### 1. Edit .env file
* Copy `.env.example` to `.env`.
* Fill in `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `CAMO_KEY`, `SECRET_KEY_BASE`, `POSTGRESS_PASSWORD`, and `EXTRACT_SECRET`. I recommend randomly generating seperate alphanumeric passwords for each of these values. Having secure passwords is especially crucial as this installation is exposed to the internet.
* Then, you should fill `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` with the same values as `MINIO_ACCESS_KEY`and `MINIO_SECRET_KEY`, respectively.
* Replace all occurences of `domain.tld` with the domain you plan on using.
* Fill in email server details for automated Feedbin emails. I have my own email server, but if you don't, there are several options to obtain an email address with SMTP access.

### 2. Copy docker-compose and Caddyfile examples to proper locations
* Copy `docker-compose.yml.example` to `docker-compose.yml`. If you want to disable a service this is the place. Also run `mkdir volumes` to create the folder where the Docker volumes are stored to by default.
* Copy `caddy/Caddyfile.example` to `caddy/Caddyfile` and update the domains, just as you did in the `.env` file.

### 3. Database Setup
Initialize the database:
```sh
docker-compose up -d feedbin-postgres feedbin-web && docker exec -ti feedbin-web rake db:setup && docker-compose down
```

If you get an error message here, please double check your `.env` file and try checking the logs for `feedbin-web` or `feedbin-postgres` by running `docker logs -f feedbin-web` or `docker logs -f feedbin-postgres`.

### 4. Minio Setup
Now that you've finished the database setup, you can launch the rest of the containers, by running:

```sh
docker-compose up -d
```

* Go to `minio.feedbin.domain.tld` and login with the keys from the `.env` file.

* Create a bucket named `feedbin` with the plus button in the bottom right hand corner.

### 5. Finish Up
* I recommend restarting the containers at this point, by running:
```sh
docker-compose down && docker-compose up -d
```
* Once the containers are up, you can check if everything is going well with `docker-compose logs -f` or `docker-compose ps`.

* Then, go to `feedbin.domain.tld` and create a new account. You're set!

* You can make yourself an admin to manage users and to view the Sidekiq web interface, by running:

```sh
docker-compose exec feedbin-web rake feedbin:make_admin[youremail@domain.tld]
```

Once you're done, you can prevent new users from registering by [modifying cour Caddy config](https://github.com/angristan/feedbin-docker/issues/3#issuecomment-700286769).

### 6. Maintenance
I recommend updating your container images each time you update your system. To do so, open the your `feedbin-docker` directory and run this command:
``` sh
docker-compose pull && docker-compose build --no-cache && docker-compose up -d
```

I am working on setting up CI so the containers will automatically build each time a commit is pushed to this repo, so users don't need to build the cotainers themselves.
