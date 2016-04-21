---
layout: post
title: Developing AGI scripts with phpagi and Docker
---

The Asterisk Gateway Interface (**AGI**) is an interface for adding functionality to Asterisk with many different programming languages.

The [Agiphlow phpagi][agiphlow-phpagi] is a PHP library that implements the AGI protocol. It is based on the [phpagi project](http://phpagi.sourceforge.net/).

In this article we will create a Docker image that will let us develop AGI scripts using the [Agiphlow phpagi][agiphlow-phpagi] library

We'll start by creating a directory to place our source:

```sh
mkdir phpagi-test
cd phpagi-test
```

Now let's create a `composer.json` file to install `agiphlow/phpagi`:

```json
{
  "require": {
    "agiphlow/phpagi": "dev-master"
  },
  "minimum-stability": "dev"
}
```

Install dependencies using composer:

```sh
composer install
```

You will now see a `vendor` directory where the `agiphlow/phpagi` library has been installed.

## Dockerfile
Now,  we'll create a docker image we can reuse for all of our `agiphlow/phpagi` scripts.
We will base our image on [dougbtv/asterisk](https://hub.docker.com/r/dougbtv/asterisk/) image which already has an Asterisk Server installed:

```Dockerfile
FROM dougbtv/asterisk
```

Then we add the following line to install php on our image:

```Dockerfile
RUN yum install -y php
```

We'll define a SIP user so we can connect to our Asterisk Server to make test calls. We can achieve this appending the follwing lines to the SIP module config file (`/etc/asterisk/sip.conf`):

```Dockerfile
[phpagi]
defaultuser=phpagi
secret=secret
context=agiphlow-phpagi
type=friend
host=dynamic
insecure=port,invite
```
In our `Dockerfile` this will look something like this:

```Dockerfile
RUN printf "\n\
[phpagi]\n\
defaultuser=phpagi\n\
secret=secret\n\
context=agiphlow-phpagi\n\
type=friend\n\
host=dynamic\n\
insecure=port,invite\n"\
>> /etc/asterisk/sip.conf
```
As you can see, we defined the user *phpagi* with the context *agiphlow-phpagi*.

Now let's create an extension we can dial in to run our agi script. We can achieve this by appending the following lines to Asterisk's extension file (`/etc/asterisk/extensions.conf`):

```
[agiphlow-phapgi]
exten => 100, 1, AGI(/phpagi/agi.php)
same  =>      n, Hangup()
```

This defines the extension *100* on the *agiphlow-phpagi* context. When we dial extension *100* from our SIP client, the AGI script `/phpagi/agi.php` will get executed.

So our `Dockerfile` will end up looking like this:

```Dockerfile
FROM dougbtv/asterisk

# Install php
RUN yum install -y php

# Define SIP user
RUN printf "\n\
[phpagi]\n\
defaultuser=phpagi\n\
secret=secret\n\
context=agiphlow-phpagi\n\
type=friend\n\
host=dynamic\n\
insecure=port,invite\n"\
>> /etc/asterisk/sip.conf

# Define extension in dialplan
RUN printf "\n\
[agiphlow-phapgi]\n\
exten => 100, 1, AGI(/phpagi/agi.php)\n\
same  =>      n, Hangup()\n"\
>> /etc/asterisk/extensions.conf
```

Let's build our image:

```sh
docker build --tag phpagi-dev .
```

## Sample AGI script
Now, let's create an AGI script using `agiphlow/phpagi`. Create the file `agi.php` with the following content:

```php
<?php

require_once __DIR__ .'/vendor/autoload.php';

use Agiphlow/PhpAgi/Agi;

// turn on debugging
$agi = new Agi(array(
    'debug' => true,
    'use_default_error_handler' => true
));

// answer the call
$agi->answer();

// play message
$agi->stream_file('hello-world');

// hangup
$agi->hangup();

```

## Run docker container

> **Note:** SIP uses a wide range of UDP ports for RTP communication. Exposing many ports using docker's `expose` option can be very slow. For the purpose of this tutorial, we will use the `--net=host` option

```sh
docker run \
  --name phpagi-test \
  --net=host \
  -v `pwd`:/phpagi \
  -d -t phpagi-dev
```

As you can see, we mounted the current directory (`pwd`) on the container's `/phpagi` directory. That's where Asterisk will look for the `agi.php` script.

## Make test calls
Use your favorite SIP client to register to the SIP Server. Remember that since we used the `-net=host` option, you should point your SIP client to the IP Address of the machine where `docker` is running.

## Debugging
To debug your script use docker logs:

```sh
docker logs -f phpagi-test
```

   [agiphlow-phpagi]: <https://github.com/agiphlow/phpagi>
