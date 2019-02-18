# Heroku Buildpack: NGINX + PageSpeed

## Versions:

- [Nginx](https://www.nginx.com/) 1.15.8
- [Google Pagespeed](https://developers.google.com/speed/pagespeed/module/) 1.13.35.2-stable

## Motivation

Some application servers (e.g. Ruby's Puma) halt progress when dealing with network I/O. Heroku's Cedar routing stack [buffers only the headers](https://devcenter.heroku.com/articles/http-routing#request-buffering) of inbound requests. (The Cedar router will buffer the headers and body of a response up to 1MB) Thus, the Heroku router engages the dyno during the entire body transfer –from the client to dyno. For applications servers with blocking I/O, the latency per request will be degraded by the content transfer. By using NGINX in front of the application server, we can eliminate a great deal of transfer time from the application server. In addition to making request body transfers more efficient, all other I/O should be improved since the application server need only communicate with a UNIX socket on localhost. Basically, for webservers that are not designed for efficient, non-blocking I/O, we will benefit from having NGINX to handle all I/O operations.

## Requirements (Proxy Mode)

* Your webserver listens to the socket at `/tmp/nginx.socket`.
* You touch `/tmp/app-initialized` when you are ready for traffic.
* You can start your web server with a shell command.

## Features

* Unified NXNG/App Server logs.
* [L2met](https://github.com/ryandotsmith/l2met) friendly NGINX log format.
* [Heroku request ids](https://devcenter.heroku.com/articles/http-request-id) embedded in NGINX logs.
* Crashes dyno if NGINX or App server crashes. Safety first.
* Language/App Server agnostic.
* Customizable NGINX config.
* Application coordinated dyno starts.

### Logging

NGINX will output the following style of logs:

```
measure.nginx.service=0.007 request_id=e2c79e86b3260b9c703756ec93f8a66d
```

You can correlate this id with your Heroku router logs:

```
at=info method=GET path=/ host=salty-earth-7125.herokuapp.com request_id=e2c79e86b3260b9c703756ec93f8a66d fwd="67.180.77.184" dyno=web.1 connect=1ms service=8ms status=200 bytes=21
```

### Language/App Server Agnostic

nginx-buildpack provides a command named `bin/start-nginx` this command takes another command as an argument. You must pass your app server's startup command to `start-nginx`.

For example, to get NGINX and Puma up and running:

```bash
$ cat Procfile
web: bin/start-nginx bundle exec puma -c config/puma.rb
```

### Setting the Worker Processes

You can configure NGINX's `worker_processes` directive via the
`NGINX_WORKERS` environment variable.

For example, to set your `NGINX_WORKERS` to 8 on a PX dyno:

```bash
$ heroku config:set NGINX_WORKERS=8
```

### Customizable NGINX Config

You can provide your own NGINX config by creating a file named `nginx.conf.erb` in the config directory of your app. Start by copying the buildpack's [default config file](config/nginx.conf.erb).

### Customizable NGINX Compile Options

See [scripts/build_nginx](scripts/build_nginx) for the build steps. Configuring is as easy as changing the "./configure" options.

You can run the builds in a [Docker](https://www.docker.com/) container:

```
$ make build # It outputs the latest builds to bin/cedar-*
```

### Application/Dyno coordination

The buildpack will not start NGINX until a file has been written to `/tmp/app-initialized`. Since NGINX binds to the dyno's $PORT and since the $PORT determines if the app can receive traffic, you can delay NGINX accepting traffic until your application is ready to handle it. The examples below show how/when you should write the file when working with Puma.

## Setup

### Existing App

Update Buildpacks to use the latest stable version of this buildpack:
```bash
$ heroku buildpacks:add https://github.com/CareerFoundry/heroku-buildpack-nginx-pagespeed
```

Update Procfile:
```
web: bin/start-nginx bundle exec puma -c config/puma.rb
```

```bash
$ git add Procfile
$ git commit -m 'Update procfile for NGINX buildpack'
```

Update Puma Config
```ruby
workers Integer(ENV["WEB_CONCURRENCY"] || 2)
threads_count = Integer(ENV['RAILS_MAX_THREADS'] || 4)
threads threads_count, threads_count

rackup      DefaultRackup
bind        "unix:///tmp/nginx.socket"
environment ENV['RACK_ENV'] || 'development'

preload_app!

on_worker_fork do
  FileUtils.touch('/tmp/app-initialized')
end

```

Push Heroku App:
```bash
$ git push heroku master
$ heroku logs -t
```

Visit App
```
$ heroku open
```

## Reference

- [https://github.com/heroku/heroku-buildpack-nginx](https://github.com/heroku/heroku-buildpack-nginx)
