+++ 
date = "2020-08-26"
title = "Poor Man's Docker Deployment"
#slug = "do" 
tags = ["docker"]
categories = ["blog"]
authors = ["csgeek"]
+++

I have had several use cases where I want to deploy things in docker for multiple reasons, but ease of deployment is a big one.  Low entry bar, less maintenance that makes docker deployment more appealing.  I don't have the resources to do a full K8 deployment and ended up with this pattern I wanted to share with others


**Assumptions**:

 1. You are somewhat familiar with [docker](www.docker.com)
 2. You have some exposure with [docker-compose](https://docs.docker.com/compose/)
 3. You know what systemd, init.d scripts are etc. 
 4. You have some exposure to nginx or a similar Proxy.

## SSL

Most services require some level of security.  This is primarily focused towards web based services, so we'll be using letsencrypt.  I usually force all traffic through SSL, though you don't have to.

If you're not familiar with [letsencrypt](https://letsencrypt.org/) and [certbot](https://certbot.eff.org/) it's the defacto way of getting a valid certificate of late (for free).  There are other sources but they're all paid services.  

I won't go into the full details on setting this up, but I'm making the assumption that you have some kind of wildcard or host based SSL that are on your local file system under some variation of these paths:


``` 
/etc/letsencrypt/live/myhost.org/fullchain.pem; /etc/letsencrypt/live/myhost.org/privkey.pem;
```

If you did buy your own cert, I'm assuming you know how to get it in a format where nginx will accept them or are able to find the information through your god like google-fu.

## NGINX

There are two approaches to this.  They each have their drawbacks and advantages.

1. Running NGINX in a container.  

The big advantage to this approach is that you don't need to expose any ports except HTTPS and HTTP.  Everything else is in the docker internal network.

Downside, is that you'll have to be using the same docker network for all services.  You also will need to ensure a certain order of operation.  Likely nginx needs to come up last in order to detect the running services.  Or wait the timeout session for service discovery to work.  I've always had better luck letting everything start first and starting the web server last.

2. Running NGINX on Host

Downside: You will need to expose the services locally on various ports at the very least on localhost.  

Upside: You can simply start nginx as a system service and not worry about docker.  They are disjoint and usually works fairly well.  

I'm using the second approach so I'll mainly be exploring what needs to be done to get that working, but keep in mind that solution 1 is perfectly valid.  


### NGINX vhost config.

This will vary on your OS, but Debian bases hosts under /etc/nginx/sites-available you'll need to create the config and have a symlink on /etc/nginx/sites-enabled/

Here's an example configuration:


```yaml
server {
	# SSL configuration
	#
	listen 443 ssl;
	listen [::]:443 ssl;


        access_log /var/log/nginx/APPNAME_access.log;
        error_log /var/log/nginx/APPNAME_error.log;

	ssl on;

	ssl_certificate /etc/letsencrypt/live/appname.myhost.org/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/appname.myhost.org/privkey.pem;
        client_max_body_size 50M;

        #log_format compression '$remote_addr - $remote_user [$time_local] '
        #                       '"$request" $status $bytes_sent '
        #                       '"$http_referer" "$http_user_agent" "$gzip_ratio"';

        #access_log log/nixie_access.log  compression;

        server_name appname.myhost.org;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host            $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Real-IP $remote_addr;
         
        }


        location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        location ~* \.(txt|log)$ {
            allow 192.168.0.0/16;
            deny all;
        }

        location ~ \..*/.*\.php$ {
            return 403;
        }


        # Block access to "hidden" files and directories whose names begin with a
        # period. This includes directories used by version control systems such
        # as Subversion or Git to store control files.
        location ~ (^|/)\. {
            return 403;
        }

}

server {
	listen 80;
	listen [::]:80;
         server_name appname.myhost.org ;


	rewrite ^ https://$server_name$request_uri? permanent;
}

```

Here's the big take away from this file.  We are hosting a new application that's exposing locally port 8080.  Any request that comes in on appname.myhost.org will get redirected to localhost:8080 and we will return the response.  

This will work great, but we do need to make sure that our docker stack is also running.


## Docker application

I won't spend too much time on this but let's assume you have a wordpress application similar to the one in the example on their [docker hub](https://hub.docker.com/_/wordpress).

This is the config example pulled from that link:


```yaml
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

if everything works as expected when you start the application via docker-compose up -d http://localhost:8080 will become accessible.

Now, what we need to do is to get it so our docker based application starts up each time our host restarts.  It would be nice if we can manage a docker app like any other service installed.

So, we're going to create a systemd start up script.

### Docker Systemd script

I'm running all my docker apps as the docker_user with limited permissions.

```
[Unit]
Description = WordPress Blog
After=docker.service
Requires=docker.service

[Service]
Type=idle
WorkingDirectory=/home/docker_user/blog
ExecStart=/usr/bin/docker-compose up
ExecStop=/usr/bin/docker-compose  stop
ExecReload =/usr/bin/docker-compose  restart
User=docker_user
GUser=docker_user
Restart=always
RestartSec=3
RestartPreventExitStatus=0
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target 

```

let's save this file in /etc/systemd/system/blog.service

we can enable the service on boot via:

```
sudo systemctl enable blog.service
```

You can also test this out manually by simply running:

```
sudo service blog {start|stop|restart}
```

At this point you should be able to get a response from:

https://appname.myhost.org, granted the behavior and hostname will vary based on what you do end up running, but if you are running wordpress, You should see the Installation wizards come up.


## Docker Check List.

1. It's always a good practice to have 

```
 restart: always

```

2. Database backups are not configured here but you should have a script or cron that will create a datadump every so often.

## Final stage

Ensure the nginx config is valid by using `nginx -t` and if it all checks out let's restart the nginx server.

```
sudo service nginx start
```

At this point even if you have a power outage or someone does a sudo reboot all your services should come back up as expected.

Naturally this still suffers from a single point of failure, but it's much easier to manage IMO, then the typical bare metal deployments.
