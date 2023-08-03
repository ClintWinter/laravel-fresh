# Basic Laravel Deployment

## Linux setup

Spin up a Digital Ocean droplet.

### SSH login key

Generate or use an existing SSH Key (locally).

```
ssh-keygen -t rsa
cat id_rsa.pub | pbcopy
```

Log in to your new droplet as root with the chosen SSH key.

Provide your key name and IP address.

```
cd ~/.ssh
ssh -i id_rsa root@<ip_address>
```

### Create a user

Create user:

1. create new user
2. add user to sudo group
3. switch user to new user
4. check sudo access of new user

```
adduser clint
usermod -aG sudo clint
su clint
sudo cat /var/log/auth.log
```

To add ssh access to the user, add the public ssh key to `/home/clint/.ssh/authorized_keys` (it won't exist, so create it).

Local:
```
cat ~/.ssh/is_rsa | pbcopy
```

Remote:
```
echo "<ssh_key>" >> ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
sudo service sshd restart
```

Open a new tab and attempt to log in as the new user.

```
ssh -i id_rsa clint@<ip_address>
```

If successful, we can revoke root login.

> Important: Check that you can log in as new user and have sudo access before disabling root access!

```
sudo vim /etc/ssh/sshd_config
```

Change `PermitRootLogin yes` to `PermitRootLogin no`.

## Project Environment

### Dependencies

We still need composer, PHP, MySQL, etc. Let's figure out how to get all that:

https://laravel.com/docs/10.x/deployment#server-requirements

#### TL;DR

There's a lot happening in the following subsections that cover each and every depedency needed to get PHP up and running. I'm dropping a reference here so you can speed through it the next time you're here:

```
$ sudo add-apt-repository ppa:ondrej/php

$ sudo apt update && sudo apt upgrade

$ sudo apt install php8.2 php-curl php-mbstring php-pear php-fpm php-zip

// check php is installed at the correct version
$ php -v
```

Crazy that most of the work that took me a long time below amounted to adding a few more extensions to the initial PHP install to round it out.

#### PHP

We don't want to have to build from source, so we can use a PPA (non-official) ubuntu package for installing PHP: https://launchpad.net/~ondrej/+archive/ubuntu/php

Typically, installing a PPA happens in 3 steps:

1. Add the PPA repo to your system so we know how to download it: `sudo add-apt-repository ppa:ondrej/php`
2. Update the packages so its up to date before downloading: `sudo apt update && sudo apt upgrade`
3. Install the package: `sudo apt install php8.2`
4. `which php`  should output `/usr/bin/php` and `php -v` should give you `PHP 8.2.X`

Here's the list of dependencies from the laravel site:
* PHP >= 8.1 - we just installed it
* Ctype PHP Extension
* cURL PHP Extension
* DOM PHP Extension
* Fileinfo PHP Extension
* Filter PHP Extension
* Hash PHP Extension
* Mbstring PHP Extension
* OpenSSL PHP Extension
* PCRE PHP Extension
* PDO PHP Extension
* Session PHP Extension
* Tokenizer PHP Extension
* XML PHP Extension

We just installed PHP. Let's now figure out how to enable all of these extensions.

Type `php --ini` to get a nice breakdown of your PHP configuration. Mine looks like this:

```
Configuration File (php.ini) Path: /etc/php/8.2/cli
Loaded Configuration File:         /etc/php/8.2/cli/php.ini
Scan for additional .ini files in: /etc/php/8.2/cli/conf.d
Additional .ini files parsed:      /etc/php/8.2/cli/conf.d/10-opcache.ini,
/etc/php/8.2/cli/conf.d/10-pdo.ini,
/etc/php/8.2/cli/conf.d/20-calendar.ini,
/etc/php/8.2/cli/conf.d/20-ctype.ini,
/etc/php/8.2/cli/conf.d/20-exif.ini,
/etc/php/8.2/cli/conf.d/20-ffi.ini,
/etc/php/8.2/cli/conf.d/20-fileinfo.ini,
/etc/php/8.2/cli/conf.d/20-ftp.ini,
/etc/php/8.2/cli/conf.d/20-gettext.ini,
/etc/php/8.2/cli/conf.d/20-iconv.ini,
/etc/php/8.2/cli/conf.d/20-phar.ini,
/etc/php/8.2/cli/conf.d/20-posix.ini,
/etc/php/8.2/cli/conf.d/20-readline.ini,
/etc/php/8.2/cli/conf.d/20-shmop.ini,
/etc/php/8.2/cli/conf.d/20-sockets.ini,
/etc/php/8.2/cli/conf.d/20-sysvmsg.ini,
/etc/php/8.2/cli/conf.d/20-sysvsem.ini,
/etc/php/8.2/cli/conf.d/20-sysvshm.ini,
/etc/php/8.2/cli/conf.d/20-tokenizer.ini
```

From this output, we can learn that our `php.ini` file is located in `/etc/php/8.2/cli/php.ini`. So we should be able to edit that to load extensions.

We can also see that it attempts to load additional ini files from `/etc/php/8.2/cli/conf.d` where it found a whole list of them. Some appear to overlap with the Laravel requirements, but not all. You'll also notice that the files are prefixed with numbers like `10` and `20`. I think I learned this while trying to setup xDebug, the numbers are used to order the files. They are loaded in order, so the order sometimes matters if extensions depend on each other. We don't want to load an extension that requires another extension that wasn't loaded yet.

The following extensions overlap between the Laravel requirements and the ini list: `ctype`, `fileinfo`, `pdo`, `opcache`, `tokenizer`. 

#### Ctype

Let's start with `ctype`. I found this when searching PHP's site for ctype: https://www.php.net/manual/en/ctype.installation.php. Essentially, it's built in and requires no customization. It also says there is no configuration directives in `php.ini`. If I search the `php.ini` file for `ctype`, nothing comes up. However, we do have this ini file for ctype, so what is that? If we `cat` the file, we see this:

```
; configuration for php common module
; priority=20
extension=ctype.so
```

All that does is make sure the built-in extension is included, so looks like we are good to go.

#### fileinfo

Fileinfo is the same story. It does, however, have a commented out line in `php.ini` for loading the extension `;extension=fileinfo`. However, it also has its own ini that may load the extension. How can we find out? Well php can output an information page that tells us the result of PHP's configurations. If we type `php -i` we can see its entire breakdown. If we type `php -i | grep fileinfo` we can see the relevant information.

```
/etc/php/8.2/cli/conf.d/20-fileinfo.ini,
fileinfo
fileinfo support => enabled
```

We see it's enabled.

#### PDO

Let's speed this up and keep grepping for the various extensions.

```
$ php -i | grep pdo

/etc/php/8.2/cli/conf.d/10-pdo.ini,

$ php -i | grep PDO

PDO
PDO support => enabled
PDO drivers =>
```

Enabled!

#### OPcache

```
$ php -i | grep opcache

...
opcache.enable => On => On
opcache.enable_cli => Off => Off
...
```

So we see it's on.

#### Tokenizer

```
$ php -i | grep -i tokenizer

/etc/php/8.2/cli/conf.d/20-tokenizer.ini
tokenizer
Tokenizer Support => enabled
```

That covers all of the extensions that had ini files. 

#### cURL

Let's see if curl is enabled. We know we have curl on the server by typing `which curl` and getting `/usr/bin/curl`. From PHP site: "...As of PHP 7.3.0, version 7.15.5 or later is required. As of PHP 8.0.0, version 7.29.0 or later is required."

```
$ curl -V

curl 7.81.0 ...
```

We're good!

However, we don't appear to have curl as an extension. No output when doing `php -i | grep -i curl`.

There's a line for `extension=curl` in the `php.ini`, but when we uncomment it, we get the following error when running any PHP commands.

```
PHP Warning:  PHP Startup: Unable to load dynamic library 'curl' (tried: /usr/lib/php/20220829/curl (/usr/lib/php/20220829/curl: cannot open shared object file: No such file or directory), /usr/lib/php/20220829/curl.so (/usr/lib/php/20220829/curl.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
```

Looks like what we need to do is install `php-curl` from the PPA as well. So we can run `sudo apt update` and `sudo apt install php-curl`. If we output `php --ini` again, we will actually see `/etc/php/8.2/cli/conf.d/20-curl.ini` in the list of ini files parsed.

Re-comment out the extension in our `php.ini` so we aren't loading it twice. Now we have this: 

```
$ php -i | grep -i curl

/etc/php/8.2/cli/conf.d/20-curl.ini,
curl
cURL support => enabled
cURL Information => 7.81.0
curl.cainfo => no value => no value
```

Voila!

#### DOM

Looks like DOM requires libxml: https://www.php.net/manual/en/dom.requirements.php

If we grep for xml in our phpinfo output, we see:

```
libxml
libXML support => active
libXML Compiled Version => 2.9.14
libXML Loaded Version => 20914
libXML streams => enabled
```

So that seems okay.

We don't see anything about `DOM` though.

The docs specify it is enabled by default without any runtime configurations. So we should be good to go 🤞.

#### Filter

Installed by default.

#### Hash

Built into PHP core.

#### Mbstring

Must be installed.

```
$ sudo apt update
$ sudot apt install php-mbstring
```

`/etc/php/8.2/cli/conf.d/20-mbstring.ini` is added to the list of loaded ini files automatically.

```
$ php -i | grep -i mbstring

/etc/php/8.2/cli/conf.d/20-mbstring.ini,
Zend Multibyte Support => provided by mbstring
Multibyte decoding support using mbstring => enabled
```

#### OpenSSL

Appears to be enabled by default.

```
$ php -i | grep -i openssl

SSL Version => OpenSSL/3.0.2
libSSH Version => libssh/0.9.6/openssl/zlib
openssl
OpenSSL support => enabled
```

#### PCRE

Appears to be enabled by default.

```
$ php -i | grep -i pcre

pcre
PCRE (Perl Compatible Regular Expressions) Support => enabled
```

#### Session

Appears to be enabled by default.

```
$ php -i | grep -i session

session
Session Support => enabled
```

#### XML

We have `libxml` for the DOM package we added earlier, but I think this `xmldiff` pecl extension is something else.

I don't think we have that one.

Since it's a pecl extension, I think we have to install pear first: `sudo apt install php-pear`.

Running that install also installed `php-xml` alongside it, which may have added some XML package for us. That may be what we needed.

#### FPM

The nginx config provided by the Laravel docs has a file path to an fpm file, but we don't have it. We need to install php-fpm via `sudo apt install php-fpm`.

#### php-zip

This package is used by composer for downloading packages.

```
$ sudo apt install php-zip
```

### NGINX

We are now out of the woods of setting up our PHP system.

Let's install Nginx.

```
sudo apt install nginx
sudo service nginx start
```

Add an nginx config file to `/etc/nginx/sites-enabled/`. E.g. `/etc/nginx/sites-enabled/example.com`. Refer to the Laravel docs for what a standard Laravel appliation nginx config file should look like.

Include it in `/etc/nginx/nginx.conf` at the bottom of the `http` block, and comment out the wildcard include.

Create a directory for the project being served. E.g. `/var/www/example.com`

Change your nameserver host from Namecheap to DigitalOcean for your domain name. That means ns1/2/3.digitalocean.com records are added to the domain on Namecheap. Then on DigitalOcean, add the A records that point from the domain to the droplet IP.

This sample server file should allow you to get up and running. Put it in `/etc/nginx/sites-enabled/example.com`.

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name *.example.com;
        root /var/www/example.com/public;

        index index.php;

        charset utf-8;

        location / {
                try_files $uri $uri/ /index.php?$query_string;
        }

        error_page 404 /index.php;

        location ~ \.php$ {
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
                include fastcgi_params;
        }

        location ~ /\.(?!well-known).* {
                deny all;
        }
}
```

### Git

Now we need to clone the actual project repo onto the server.

To do that, we need to be authorized to do so from the server.

#### Option 1: new SSH key on server

We need to generate another SSH key and register it with our Github account in order to allow us to clone it.

```
ssh-keygen -t rsa
```

`cat` the `id_rsa.pub` file and copy the contents into a new ssh key in your github profile: https://github.com/settings/keys

That works as-is from my perspective, although there are other things you may want to do. Refer to Github's docs on SSH keys to do other stuff.

#### Option 2: SSH Agent forwarding

Alternatively, instead of creating a new key on every server, we can use our existing *local* key via [agent forwarding](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/using-ssh-agent-forwarding). When you ssh into a server, you can share your local keys with the host. All you have to do is add this to `~/.ssh/config` (just replace example.com with the domain or IP of your server):

```
Host example.com
  ForwardAgent yes
```

To test if it works, I renamed `id_rsa` to `id_foo` and ran `ssh -T git@github.com` (which tests the SSH key). It failed.

Then I performed the above step and added the server IP to my config file. When I re-SSH'd into the server, it worked again.

Let's now clone in our repo and set up proper permissions.

```
$ cd /var/www/example.com
$ git clone git@github.com:Username/repository.git .
$ cd ..
$ sudo chown www-data:www-data -R example.com
```

If we don't use the `www-data:www-data` user/group, we are likely to hit a permissions error when visiting the app.

### Laravel project setup

Next we need to setup our actual Laravel application.

#### Composer

First, we need to install [composer](https://getcomposer.org/download/). Follow the instructions for installing it for your user (doesn't require `sudo`): 

* `mkdir -p ~/.local/bin/`
* `mv composer.phar .local/bin/composer`
* Use `vim .bashrc` to add the line `export PATH="$PATH:$HOME/.local/bin"`
* `source .bashrc` to reload the config
* `composer` to see if compose is now an accessible command

Before we install with composer, there's another PHP extension that is useful:

Next, run `composer install --no-dev -o` to install composer packages for production.

#### Env

Now we need to create a `.env` file. Clone `.env.example` via `cp .env.example .env`.

Next, generate an application key with `php artisan key:generate`, which will automatically update the `.env` file.

Turn on [maintenance mode](https://laravel.com/docs/10.x/configuration#maintenance-mode) with a secret key: `php artisan down --secret="my-secret-key"`.

Visit `example.com/my-secret-key` and you should now see the Laravel homepage (or whatever your `/` route is)!

That should pretty much do it for getting started with setting up a Laravel application.

The rest of the dependencies are up to you to setup.

Typically you'll need a database, a queue driver, cacheing mechanism, etc.

 ## Conclusion

Well... that was a heck of a lot of work, huh?

Let's recap.

We spun up a server on DigitalOcean.

We enabled login via SSH.

We disabled root login and created a user for security.

We set up our project environment by installing PHP and Nginx.

We connected to Github and cloned in our project.

Then we set up the project by creating an env file and installing composer and the project's dependencies.

And this is just the beginning. We haven't event set up all of the typical Laravel dependencies like a database, cache, queue, mail, monitoring, bug logging, and more.

There's also more we can do:

* We can take steps to make the server more secure like using a firewall to ensure unused ports are disabled.
* We can do primitive CI/CD to update the project when master is updated.
* We can ensure we keep the server up-to-date and healthy automatically.

It's no wonder cloud and container solutions have become so popular. Doing this stuff is annoying and kind of a waste of your precious time as someone who wants to build things.
