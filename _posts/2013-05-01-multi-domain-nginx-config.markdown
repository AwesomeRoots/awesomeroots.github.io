---
published: true
title: Multi-domain nginx config
layout: post
---
Okay, so we use nginx all the way here.

Also we often get some code to "take a look" at and "play around" with. Normally it goes like that:

1. Deploy the code.
2. lnspect its structure and find out if it's similar to one of our beloved framework.
3. If so, use the known config for this framework.
4. If not, try to find out which framework the site is based on (docs? never got a documented code for maintenance yet, huh) and google for a config for this framework.
5. If no framework is used at all (hello, spaghetti code!) use some custom config.

At some point we were tired by configuring the web server for each new site and wanted a single config, which could probably serve most PHP-based sites and frameworks.

## Why not apache?

Well, `mod_vhost_alias` + `mod_rewrite` could solve most of the things described below in default configuration.

However, we use nginx in all the development projects — it's simple, fast, secure and scalable. Yet we believe that development environment should be always the same as production environment. This policy helps us to avoid tons of related bugs.

That's why we use nginx since the initial deployment.

## So, nginx

The config of our dream will be:

* multidomain (set up once, use for any site);
* able to serve static content;
* able to serve most PHP applications (including those with single front controller as well as with multiple per-page front controllers);
* able to trim /app.php/ (Symfony) and /index.php/ (most of other frameworks);
* as a bonus, will be able to ignore www. prefix (easy done).

Voilà:

```nginx
server {
    listen  80;

    server_name ~^(www\.)?(?<domain>.+).base.domain$;

    root    www/$domain/;

    rewrite ^/app\.php/?(.*)$ /$1   redirect;
    rewrite ^/index\.php/?(.*)$ /$1 redirect;

    location / {
        try_files   $uri $uri/index.html @site;
    }

    location @site {
        if (-f $document_root/index.php) {
            rewrite ^(.*)$ /index.php/$1 last;
        }
        if (-f $document_root/app.php) {
            rewrite ^(.*)$ /app.php/$1 last;
        }
        return  404;
    }

    location ~ \.php {
        include fastcgi_params;

        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_pass            127.0.0.1:9000;
        fastcgi_param           SCRIPT_FILENAME $document_root/$fastcgi_script_name;
    }
}
```

Usage:

* deploy the config;
* replace base.domain by any base domain that you use;
* update root;
* symlink your sites webroots into the root.

## QA note

This config is not meant be anyhow secure by design.
This config is not meant to be fast and scalable.

Please use this config only for development and don't use it on production machines as is.
