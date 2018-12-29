# Setting up Grav with Nginx 

I am running a Amazon EC2 instance. System config:
Run `lscpu`:
```
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              1
On-line CPU(s) list: 0
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz
Stepping:            2
CPU MHz:             2400.125
BogoMIPS:            4800.12
Hypervisor vendor:   Xen
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            30720K
NUMA node0 CPU(s):   0
```

OS info:
Run `cat /etc/os-release`: In my case it is as:
```
NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
...
```

Notice the `ID_LIKE` key. So we need to follow the CentOS line of cmds.


# Setup PHP:

I am using `PHP 7.2.11 `. Install php along with the necessary modules:



```
# yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm

# yum install yum-utils

# yum-config-manager --enable remi-php72 

# yum install php php-mbstring php-fpm php-mcrypt php-cli php-gd php-curl php-mysql php-ldap php-zip php-fileinfo 
```

Check your php version with `php -v`. Several other modules will be necessary. Here is what my `php -m` returns:

```
[PHP Modules]
bz2
calendar
Core
ctype
curl
date
dom
exif
fileinfo
filter
ftp
gd
gettext
hash
iconv
json
libxml
mbstring
mysqli
mysqlnd
openssl
pcntl
pcre
PDO
pdo_mysql
pdo_sqlite
Phar
readline
Reflection
session
SimpleXML
sockets
SPL
sqlite3
standard
tokenizer
wddx
xml
xmlreader
xmlwriter
xsl
zip
zlib

[Zend Modules]
```

You MUST install the `php-fpm` module (in case you missed it). It is required for running Nginx.

The `mbstring` module is super important to run Grav. If you missed/forgot to install any modules, Grav will throw error once you run: `./bin/grav install`


# Setup Nginx:

Install nginx, I had to follow the CentOS steps:

```
# sudo yum install epel-release
# sudo yum install nginx
```
Start Nginx:
```
# sudo systemctl start nginx
```

By default the server root directory will be: `/usr/share/ngnix/html/`

Runing your webroot will show the default Nginx welcome page. In case you are already running apache, make sure to stop it.

```
# sudo systemctl stop apache2
```

Then restart nginx:

```
# sudo systemctl restart nginx
```

The default ngnix config file is located in `/etc/nginx`. For setting up Grav I changed it's contents with the one given in their [docs](https://learn.getgrav.org/webservers-hosting/servers/nginx#example-nginx-conf). 

```
# cat /etc/nginx/nginx.conf
```

Mine shows like this:


```
user nginx;
worker_processes auto;
worker_rlimit_nofile 8192; # should be bigger than worker_connections
pid /run/nginx.pid;

events {
    use epoll;
    worker_connections 8000;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 30; # longer values are better for each ssl client, but take up a worker connection longer
    types_hash_max_size 2048;
    server_tokens off;

    # maximum file upload size
    # update 'upload_max_filesize' & 'post_max_size' in /etc/php5/fpm/php.ini accordingly
    client_max_body_size 32m;
    # client_body_timeout 60s; # increase for very long file uploads

    # set default index file (can be overwritten for each site individually)
    index index.html;

    # load MIME types
    include mime.types; # get this file from https://github.com/h5bp/server-configs-nginx
    default_type application/octet-stream; # set default MIME type

    # logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # turn on gzip compression
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    # disable content type sniffing for more security
    add_header "X-Content-Type-Options" "nosniff";

    # force the latest IE version
    add_header "X-UA-Compatible" "IE=Edge";

    # enable anti-cross-site scripting filter built into IE 8+
    add_header "X-XSS-Protection" "1; mode=block";

    # include virtual host configs
    include sites-enabled/*;
}
```
NOTICE the `user` key on the top is set to `nginx`. This is super important. I had to set up user:group for the webroot directories as `nginx:nginx`

Now, I have set my directory to serve files as `/var/www/`. Which basically means the nginx server will lookup files inside this directory in order to serve.

File permissions can be a pain in the ass. Make sure you set up the same user as user:group in all the directories/files you want to serve. The user should be the same as the one mentioned in the `nginx.conf`

Change it quickly:

```
# sudo chown -R nginx:nginx /var/www/
```

Now clone a fresh installation of grav inside `/var/www/`

```
# git clone https://github.com/getgrav/grav.git
```

Next, I am following the grav site config from [here](https://learn.getgrav.org/webservers-hosting/servers/nginx#grav-site-configuration). 

Basically grav provides a config file for each server setup (mostly). It is in `./grav/webserver-configs/`. For nginx we want the `nginx.conf` inside it.

We will copy it inside the `sites-available`. I didn't have the directories `sites-available` and `sites-enabled`, so just made:

```
# cd /etc/nginx
# sudo mkdir sites-available
# sudo mkdir sites-enabled
```

Now do this:


```
# sudo cp /var/www/grav/webserver-configs/nginx.conf /etc/nginx/sites-available/grav-site
```

Create a symbolic link in the `sites-enabled`:

```
ln -s /etc/nginx/sites-available/grav-site /etc/nginx/sites-enabled/grav-site
```

Now, in my case the `grav-site` looks like this:

```
server {
    #listen 80;
    index index.html index.php;

    ## Begin - Server Info
    root /var/www/;
    server_name <IP/DOMAIN>;
    ## End - Server Info

    ## Begin - Index
    # for subfolders, simply adjust:
    # `location /subfolder {`
    # and the rewrite to use `/subfolder/index.php`
    location /cova-blog {
        try_files $uri $uri/ /cova-blog/index.php?$query_string;
    }
    ## End - Index

    ## Begin - Security
    # deny all direct access for these folders
    location ~* /(\.git|cache|bin|logs|backup|tests)/.*$ { return 403; }
    # deny running scripts inside core system folders
    location ~* /(system|vendor)/.*\.(txt|xml|md|html|yaml|yml|php|pl|py|cgi|twig|sh|bat)$ { return 403; }
    # deny running scripts inside user folder
    location ~* /user/.*\.(txt|md|yaml|yml|php|pl|py|cgi|twig|sh|bat)$ { return 403; }
    # deny access to specific files in the root folder
    location ~ /(LICENSE\.txt|composer\.lock|composer\.json|nginx\.conf|web\.config|htaccess\.txt|\.htaccess) { return 403; }
    ## End - Security

    ## Begin - PHP
    location ~ \.php$ {
        # Choose either a socket or TCP/IP address
        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
        # fastcgi_pass unix:/var/run/php5-fpm.sock; #legacy
        # fastcgi_pass 127.0.0.1:9000;

        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
    }
    ## End - PHP
}
```

NOTICE here - I want to serve from `/var/www/cova-blog/`, so there are a few changes I did:

* The root is given as the root directory where nginx should serve:
```
 root /var/www/;
```

* Since I want to serve from a sub-directory, I had to point it in the location key:

```
 location /cova-blog {
        try_files $uri $uri/ /cova-blog/index.php?$query_string;
    }
```

* You can either put your IP or Domain in the `server_name` key

Restart Nginx:

```
# sudo systemctl restart nginx
```


# Gotchas

