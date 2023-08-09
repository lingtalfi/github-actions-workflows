# Laravel - deployment steps

This is my memo to deploy a laravel app on a vps, using docker.

## Overview - the actors

- the vps, with nginx installed as a reverse proxy
- two containers:
  - the app container (running the laravel app) coupled with its own nginx webserver
  - the db container, running mysql

Why coupling laravel app with a nginx webserver?
To avoid complications when dealing with file uploaded by users on the laravel app. I just upload them in the public folder of the laravel app, and nginx serves them directly.

## The nginx reverse proxy

The nginx reverse proxy on the vps is used to route the requests to the right container.
It also handles the ssl certificate.
The first block redirects all http requests to https.
All traffic on port 443 is routed to the local port of the app container (8004 in my case).

```nginx
server {
    listen 80;
    server_name larablog.ling-underground.fr;
    return 301 https://$host$request_uri;
}


server {
    listen 443 ssl;
    server_name larablog.ling-underground.fr;

    ssl_certificate /etc/letsencrypt/live/ling-underground.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ling-underground.fr/privkey.pem;

    location / {
        proxy_pass http://localhost:8004;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

So now that we have the reverse proxy, we need to make our app container listen on port 8004.

## Important presteps

Before tackling the github action workflow, i recommend do a few things.
Those are specific steps to laravel, and they need to be done for the app to work properly.

- force the traffic to be https, i used this:

```php
    // in app/Providers/AppServiceProvider.php
    public function boot(): void
    {
        if (app()->environment('production')) {
            URL::forceScheme('https');
        }
    }
```

- In app/Http/Middleware/TrustProxies.php, change this:

```php
    protected $proxies;
```

to this:

```php
protected $proxies = '*';
```

I needed this to basically allow file uploads in the filament admin.

Took me a long time to find those two steps, so i hope it will save you some time.

## The github action workflow

The github workflow you will find in [here](../.github/workflows/build-push-deploy-prod-laravel.yml).

This workflow is triggered when a tag ending with the "-prod" suffix is pushed to the repo.
It does the following:

- builds the app image and pushes it to the docker hub
- connects to the vps via ssh, and creates the docker-compose.yml file
- connects to the vps via ssh, pulls the new image, and use the docker-compose.yml file to run it
- calls a cleanup script on the vps
- optionally (commented last step), empties the docker-compose.yml for security reason

Some notes:

- when building the app image, i actually pass the content of the .env file as a base64 string. That's to circumvent a limitation of github secrets which doesn't accept multiline values. The .env file is then created in the container from the base64 string (see the Dockerfile content for more info).

Note: to build a base64 string from a file, you can use the following command (at least on mac):

```bash
base64 .the.env.production > .env_base64
```

- [The Dockerfile](apps/laravel/Dockerfile) is pretty straightforward. It just installs php, composer, and the php extensions needed by laravel. It also copies the nginx config file and a few other things.
- [The nginx conf](apps/laravel/nginx.docker.conf) has nothing special, except two aliases for the build directory (built with npm build command), and to serve the uploaded files directly from the public folder of the laravel app (storage alias).

## The cleanup script

```bash
#!/bin/bash

# Name of the Docker container
CONTAINER_NAME="larablog-laravel.test-1"

# Command to run inside the container
COMMAND="php artisan config:clear && php artisan route:clear && php artisan view:clear && php artisan app:reset-app"

# Execute the command inside the specified container
docker exec $CONTAINER_NAME sh -c "$COMMAND"

# Check for success
if [ $? -eq 0 ]; then
  echo "App reset command and cache clearing executed successfully."
else
  echo "Failed to execute app reset command and cache clearing."
  exit 1
fi
```

As you can guess, i clear the caches, and then i call a custom artisan command to reset the app. This is not mandatory, but i use this command also with a cron job to reset the app (database and images) every hour or so.
