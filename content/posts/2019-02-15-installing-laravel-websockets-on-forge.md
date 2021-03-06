---
title: Laravel Websockets on Forge
slug: installing-laravel-websockets-on-forge
aliases:
  - /installing-laravel-websockets-on-forge
author: Alex Bouma
tags:
  - laravel
  - php
  - guide
date: 2019-01-15T12:28:00+01:00
description: "Learn how to run the Laravel Websockets package in Laravel and deploying it to a Forge provisioned server with SSL."
draft: false
cover: /img/ghost/unsplash-code-ws.jpg
---

It seems the documentation and or explanation on how to use [Laravel Websockets](https://docs.beyondco.de/laravel-websockets/) on Forge with SSL is not sufficient or clear enough and some have asked for an in-depth overview on how to install Laravel Websockets in Laravel and deploy it to Forge with SSL (be it via Let's Encrypt or Cloudlfare).

In this post I'll try my best to explain how I have done it, but keep in mind I am using a new Laravel project which has no real code in it. But the gist should be the same for all projects and should help you get set up. Let's jump in!

_Note: I am assuming a few things in this "guide". That the reader knows how to use composer, install Laravel (and packages) and how to setup a Forge server (with Daemons, firewall rules and a Let's Encrypt certificate)._

## Setting up

I am using the Laravel installer to install a new Laravel installation. The instructions on how to do that are in the [Laravel Documentation](https://laravel.com/docs/5.7/installation#installing-laravel).

After running `laravel new laravel-websockets-example` on my local machine I jump right in and install the Laravel Websockets package using `composer require beyondcode/laravel-websockets` and following the [installation guide](https://docs.beyondco.de/laravel-websockets/1.0/getting-started/installation.html) to publish the migrations and the configuration file.

I add a little boilerplate code to the `welcome.blade.php` file so we can use that later to verify everything works:

```html
<div class="flex-center position-ref full-height">
    <div class="content">
        <div class="title m-b-md">
            Laravel Websockets
        </div>

        <div class="links">
            Currently <span id="online">0</span> people are viewing this page!
        </div>
    </div>
</div>
```

I also added the following configuration settings to my .env file:

```
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=PFKJ5W3TYTFnv7p5yVRWsBPd
PUSHER_APP_KEY=wttTXkwAPaP8pu2M25MFNv2u
PUSHER_APP_SECRET=4czbE8JbHNDRSZTUSGdEw9QQ
```

These values are used in the `websockets.php` configuration file and also in the `broadcasting.php` file.

_Note: Do not change any other values in `websockets.php`. You might think you need to configure an SSL certificate file in there, you don't! We use NGINX for SSL termination, that means NGINX handles the SSL part and forwards plain HTTP traffic to the websockets server, more about that later._

I add the following connection to the `broadcasting.php` file, this assumes I am running my websocket server on port `6001` on the same machine as the application is running (which is the default and it will be in this example):

```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'encrypted' => true,
        'host' => '127.0.0.1',
        'port' => 6001,
        'scheme' => 'http'
    ],
],
```

I then proceed to install Laravel Echo following the [official documentation](https://laravel.com/docs/5.7/broadcasting#receiving-broadcasts) my Echo looks like this:

```js
window.Echo = new Echo({
    broadcaster:       'pusher',
    key:               window.PUSHER_APP_KEY,
    wsHost:            window.location.hostname,
    wsPort:            window.APP_DEBUG ? 6001 : 6002,
    wssPort:           window.APP_DEBUG ? 6001 : 6002,
    disableStats:      true,
    encrypted:         !window.APP_DEBUG,
    enabledTransports: ['ws', 'wss'],
});
```

Important is to note I added `wssPort` and set `encrypted` to whatever `APP_DEBUG` is not since we want that SSL goodness but not locally.

You'll notice I get the app key from `window.PUSHER_APP_KEY` and the debug state from `window.APP_DEBUG` I have set that in `welcome.blade.php` using the following snippet (before I load my JavaScript). It reads the app key from the broadcasting connection we setup earlier.

```html
<script>
    window.PUSHER_APP_KEY = '{{ config('broadcasting.connections.pusher.key') }}';
    window.APP_DEBUG = {{ config('app.debug') ? 'true' : 'false' }};
</script>
```

I proceed to add a proof of concept piece of code that will just simply count all users in a presence channel.

```js
var onlineUsers = 0;

function update_online_counter() {
    jQuery('#online').text(onlineUsers);
}

window.Echo.join('common_room')
    .here((users) => {
        onlineUsers = users.length;

        update_online_counter();
    })
    .joining((user) => {
        onlineUsers++;

        update_online_counter();
    })
    .leaving((user) => {
        onlineUsers--;

        update_online_counter();
    });
```

This is about all the setup needed to get our example application up and running!

Quick reminder: the stuff I did in my `BroadcastServiceProvider` is not how it's meant to be! Presence are supposed to be authenticated but for this example I did not want to scaffold a user login so I took my question to Google and found some guidance on a [Stack Overflow](https://stackoverflow.com/questions/43341820/laravel-echo-allow-guests-to-connect-to-presence-channel) post and adapted it for our use case. This allows an unauthenticated `common_room` channel that everyone and their dog can join. Do not do this in you app please!

## On to the Forge!

So the next step will be deploying this application to Forge, I uploaded my application to [GitHub](https://github.com/stayallive/laravel-websockets-example). I also created a site on a Forge server for it and deployed it using the Forge apps feature. After it was deployed I added a Let's Encrypt certificate. This can all be done from the Forge interface and requires no SSH (Forge is great!).

There are two ways to proceed, the first is to use the same domain as the application but a different port than 443 (which we will do) or use a separate (sub-)domain.

The first is easier since it does not require any extra setup except for some additions to the NGINX configuration (which is only some copypasta, which is about 99% of [your job](https://i.imgur.com/wOsEq7N.png) anywho).

The second way requires you to create a separate domain (or site in Forge terms) for the socket server to live on which allows you to run on it on an entirely different server or run it on port `443` if that's your jam.

We will use the domain your app is hosted on and make the websocket server available on a different port.

To do this edit the NGINX configuration for your site in Forge and do the following. Duplicate the server block  that is in there and modify the port from `443` to `6002` (why not `6001` will be explained later) and replace the `location / { /* ... */ }` block with the configuration from the [Laravel Websockets documentation](https://docs.beyondco.de/laravel-websockets/1.0/basic-usage/ssl.html#usage-with-a-reverse-proxy-like-nginx).

If you did that the NGINX config should look a little something like this:

```nginx
# FORGE CONFIG (DO NOT REMOVE!)
include forge-conf/laravel-websockets-example.alexbouma.me/before/*;

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name laravel-websockets-example.alexbouma.me;
    root /home/forge/laravel-websockets-example.alexbouma.me/public;

    # FORGE SSL (DO NOT REMOVE!)
    ssl_certificate /etc/nginx/ssl/laravel-websockets-example.alexbouma.me/473558/server.crt;
    ssl_certificate_key /etc/nginx/ssl/laravel-websockets-example.alexbouma.me/473558/server.key;

    ssl_protocols TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/dhparams.pem;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    # FORGE CONFIG (DO NOT REMOVE!)
    include forge-conf/laravel-websockets-example.alexbouma.me/server/*;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/laravel-websockets-example.alexbouma.me-error.log error;

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

server {
    listen 6002 ssl;
    listen [::]:6002 ssl;
    server_name laravel-websockets-example.alexbouma.me;
    root /home/forge/laravel-websockets-example.alexbouma.me/public;

    # FORGE SSL (DO NOT REMOVE!)
    ssl_certificate /etc/nginx/ssl/laravel-websockets-example.alexbouma.me/473558/server.crt;
    ssl_certificate_key /etc/nginx/ssl/laravel-websockets-example.alexbouma.me/473558/server.key;

    ssl_protocols TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/dhparams.pem;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    # FORGE CONFIG (DO NOT REMOVE!)
    include forge-conf/laravel-websockets-example.alexbouma.me/server/*;

    location / {
        proxy_pass             http://127.0.0.1:6001;
        proxy_read_timeout     60;
        proxy_connect_timeout  60;
        proxy_redirect         off;
        
        # Allow the use of websockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/laravel-websockets-example.alexbouma.me-error.log error;

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

# FORGE CONFIG (DO NOT REMOVE!)
include forge-conf/laravel-websockets-example.alexbouma.me/after/*;
```

_Note: [the docs document](https://docs.beyondco.de/laravel-websockets/1.0/basic-usage/ssl.html#usage-with-a-reverse-proxy-like-nginx) a methods on how to run the websockets server on the same domain as your application but on a different path but I personally don't like that since it can collide with your app routes. However, it's an option!_

Next is opening up port `6002` in the firewall so it's reachable from the internet. You can do that from the Network tab of your Forge server and add a rule called "Websockets" or something descriptive and port `6002` and leave the IP's field blank (all IP's are allowed to connect to this port).

After this there is just 1 thing left to do! Start up the websockets server as a daemon so it is automatically restarted in case it crashed or the server reboots!

To do that in Forge go to the Daemons tab of your server and add a new daemon with the command `php artisan websockets:serve` and set the directory to wherever your site is located on the server, in my case this is `/home/forge/laravel-websockets-example.alexbouma.me`. If you are deploying with [Envoyer](https://envoyer.io)/[Deployer](https://deployer.org/) or something similair you might need a path like `/home/forge/laravel-websockets-example.alexbouma.me/current`to point to the current path of your application.

_Note: This daemon is not restarted each time your app is deployed and that's probably not a good idea (because every restart of the websockets server disconnects all clients) but keep in mind that new app key's or updates of the Laravel Websockets package do require you to restart the daemon to take effect, something to keep in mind!_

For completeness, to go with a separate (sub-)domain for your socket server to be served on you can add a new site (for example `socket.yourapp.com`) and the only change you need to make to that site after enabling Let's Encrypt (you don't even need to deploy your app code to the site, it's just a configuration placeholder) is to replace the location block in the NGINX config with the one from the [Laravel Websockets documentation](https://docs.beyondco.de/laravel-websockets/1.0/basic-usage/ssl.html#usage-with-a-reverse-proxy-like-nginx). And use port `443` instead of `6002` in your Pusher/Echo client.

## But why port `6000`, `6002` and `433`, what a mess!

I hear ya! Let me explain a bit, it will hopefully all make sense afterwards.

Here is the thing, opening an port on your server can only be done by only one application at a time (technically that is not true, but let's keep it simple here). So if we would let NGINX listen on port `6001` we cannot start our websockets server also on port `6001` since it will conflict with NGINX and the other way around, therefore we let NGINX listen on port `6002` and let it proxy (NGINX is a reverse proxy after all) all that traffic to port `6001` (the websockets server) over plain http. Stripping away the SSL so the websockets server has no need to know how to handle SSL.

So NGINX will handle all the SSL magic and forward the traffic in plain http to port `6001` on your server where the websockets server is listening for requests.

The reason we are not configuring any SSL in the `websockets.php` config and we define the `scheme` in our `broadcasting.php` as `http` and use port `6001` is to bypass NGINX and directly communicate with the websockets server locally without needing SSL which faster (and easier to configure and maintain).

## A note about ports

Websockets are not allowed to connect on any port you can think of... as [Stack Overflow](https://stackoverflow.com/a/4314070/1580028) found out a lot of them are blocked (by browsers) and while testing I used port `6000`, which also seems blocked, that why I used port `6002` which works just fine.

I had a hard time figuring out what was going on since no visible errors are thrown when you use a port that is blocked by the browser :( But looking at the pusher events I saw the error which mumbled about the port I choose not being allowed for a websockets connection.

You can see the events of the Pusher client by running `window.Echo.connector.pusher.connection.timeline.events` in the developer console and inspecting the entries.

## Just show me the code already!

Here it is: [https://github.com/stayallive/laravel-websockets-example](https://github.com/stayallive/laravel-websockets-example)And here it's running: [https://laravel-websockets-example.alexbouma.me](https://laravel-websockets-example.alexbouma.me)

The full git history is there of things I have done and installed so you can see what happened and what I did, it's not that much really.

## Wait! What about Cloudflare?

What about it? :)

No really, it's a easy change make the websocket work when you are behind Cloudflare (with the orange icon enabled). Do be aware they have ["limits"](https://support.cloudflare.com/hc/en-us/articles/200169466-Can-I-use-Cloudflare-with-WebSockets-) on the volume of connections that you might hit, although they seem to handle overuse pretty nice and not just blackhole your site.

Cloudflare [lists some http(s) ports you are allowed to use](https://support.cloudflare.com/hc/en-us/articles/200169156-Which-ports-will-Cloudflare-work-with-) other than `443` for SSL. So changing port `6002` in the examples above to port `2053` will result in a working environment behind the Cloudflare proxy. Make sure you are using an https port from that list if you want it to work with SSL, so for example `2053` not `2052` since it does not support SSL. All ports support websockets.

I used an origin certificate instead of one from Let's Encrypt on my app in Forge and changed the ports mentioned above (not forgetting to open the correct one in the firewall) and it just works (tm).

See the git changes: [https://github.com/stayallive/laravel-websockets-example/tree/cloudflare](https://github.com/stayallive/laravel-websockets-example/tree/cloudflare) (I changed 6001 to 6003 to run 2 websocket servers on the same server, no need to do this yourself to make it work)And see it working: [https://laravel-websockets-example.bouma.me/](https://laravel-websockets-example.bouma.me/)

## Let me know!

I hope this helped with setting up your own websockets server on Forge in your Laravel app. [Let me know](https://twitter.com/stayallive) how it went and where you got stuck/unstuck.

