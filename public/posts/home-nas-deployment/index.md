# Home NAS deploment


.. contents::

===================
**Part I - Basic**
===================

Before All...
================

1. What is a home NAS?

A typical home NAS is used for storing and sharing photos and files.
As an extension to traditional NAS, the server config followed by this article would include more applications like offline downloader, media server, and printer server...

2. Hardware requirement:

Any kind of popular architecture (x86, amd64, arm64) CPU, with high-speed network (ethernet, WIFI) and disk (USB3.*, SATA, NVME) interface. I&#39;m not going to stop you from using an Android phone with a type-C to 4*USB (with PD backwards charging) as your cherished server.

Because technically I used such a server for more than 6 months, a Raspberry Pi 4. I had developed a habit to reboot it regularly every 2 days and bear the risk of losing all data.

So, be kind to yourself at this stage. Power and noise worth be considered as part of the budget, and also extension capability.

This is my current hardware, just for reference:

::

  cpu:
                        Intel(R) Celeron(R) N5105 @ 2.00GHz, 2800 MHz
  graphics card:
                        Intel JasperLake [UHD Graphics]
  disk:
    /dev/nvme0n1         KIOXIA NVMe SSD  # root disk
    /dev/sda             TOSHIBA MG08ADA8 # data disk
  memory:
                        Memory Size: 15 GB

Choice of Operating System
==========================

**Linux**

Not a nerd? Go for `Synology NAS &lt;https://www.synology.com&gt;`_

About File System
=================

I&#39;m currently using Btrfs, which is a native Linux file system that has lots of modern features attracts me:

1. CoW (copy on write): File won&#39;t be overwritten partially and then broken after a sudden power outage
2. Compression: save your disk and money
3. Subvolume and snapshot: save your data from accident such as you deleted your system (you won&#39;t, right?)

How to install Linux
====================

I believe there are thousands of tutorials for beginners.

`Join the Arch &#34;cult&#34; &lt;https://wiki.archlinux.org/title/Installation_guide&gt;`_

`I don&#39;t trust any binary distro, I want to compile everything, where should I start? &lt;https://wiki.gentoo.org/wiki/Handbook:Main_Page&gt;`_

`I just want a normal and stable one &lt;https://www.debian.org/releases/stable/installmanual&gt;`_


=====================
**Part II - Network**
=====================

 .. note::
 
	2024.2.11 Update:

	I switched to use **cloudflare** as my domain host, they provided some useful features:
	
	1. I can create a tunnel between cloudflare server and my server, which means when I enable cloudflare WARP on my client, the delay can very much reduce to 50ms.

	2. DDoS-protection, although I don&#39;t think there is any point to attract a personal Blog.

.. image:: /images/NAS-network.svg

As far as I know, many ISPs (Internet Service Provider) block **Port** 80 and 443, which basically means you can not access directly to the web page.
There are many workarounds: use another port instead and specify it when you access; use IPv6 instead but 80 and 443 may still be blocked; use a reverse proxy server but with higher delay.

I&#39;m using `frp &lt;https://github.com/fatedier/frp&gt;`_ as my reverse proxy software. A reverse proxy server means all your data will pass through a server held by others, so for those highly concerned about security, check out my previous post `here &lt;https://bajzc.com/posts/lets-encrypt-gentoo/&gt;`_

**Frpc** (client side) listens to a server at a specific port, your DNS record should point to that server. Once a request is made, the server will pass it to you according to the domain name, then frpc will forward it to the right application.

To be clear, frpc is still an application behind the firewall and inside docker. So if you are using IPv4 only, you can block all ports other than the one frpc is using. (only TCP)

Here is my frpc config file:

::

  [common]
  server_addr = frp2.freefrp.net  # proxy server you are using
  server_port = 7000              # port given by the server holder, default: 7000
  token = freefrp.net             # password, here the server is for public: https://freefrp.net/

  [ipv4_bajzc_com_http]           # a unique entity
  type = http                     # proxy type
  local_ip = 10.5.0.3             # local server address
  local_port = 80                 # local server port
  custom_domains = ipv4.bajzc.com # URL you use to access this application

  [ipv4_bajzc_com_https]
  type = https
  local_ip = 10.5.0.3
  local_port = 443
  custom_domains = ipv4.bajzc.com

  ......

A single frpc cannot stand, you need a **frps** (server) and correct DNS records.
A frp server could be a VPS you buy from big cloud computing company, or from **&#34;kind&#34;** people share their resources.
(Again, people concerned about security should check out my previous post `here &lt;https://bajzc.com/posts/lets-encrypt-gentoo/&gt;`_)

My DNS records as reference:

.. figure:: /images/DNS-records.png
  :align: center

  DNS-records

=====================
**Part III - Docker**
=====================
Suppose you overcome all difficulties and get your network and disk working now.

This article is mainly focusing on a Linux environment. Thanks to the portability of `Docker &lt;https://www.docker.com/&gt;`_, it could also apply to a Windows server, but comes at the expense of performance.

.. figure:: /images/Docker-structure.svg
  :align: center

  Docker applications structures

Here, address in yellow show the IP behind local subnet, which can only be access by local applications.
The NPM (nginx proxy manager) is used to handle all access for all domains and warp them with HTTPS.

Docker Network:
===============

.. code-block:: yaml

    networks:
      redisnet:                         # cache for Nextcloud
      NPM:                              # all apps should be under this subnet
        driver: bridge
        ipam:
          config:
            - subnet: 10.5.0.0/16
      qb-ipv6:                          # for IPv6 download
        enable_ipv6: true
        driver: bridge
        ipam:
          driver: default
          config:
            - subnet: 2001:3200:3200::/64
              gateway: 2001:3200:3200::1

Frpc
====

.. code-block:: yaml

    frp:
      restart: always                           # auto start after a reboot
      image: snowdreamtech/frpc:0.51.0
      container_name: frpc
      network_mode: &#34;host&#34;                      # use the host network
      volumes:
      # mount the config file dir into docker container (read only)
        - /all-docker-data/frp:/etc/frp:ro


Nginx Proxy manager
===================

.. code-block:: yaml

    NPM:
      image: jc21/nginx-proxy-manager:latest
      container_name: nginx-proxy-manager
      restart: unless-stopped
      networks:
        NPM:
          ipv4_address: 10.5.0.3                        # address frpc should point to
      ports:
    #      - &#34;0.0.0.0:81:81&#34;                            # Admin WebUI, disable after setup
        - &#34;[::]:81:80&#34;                                  # For IPv6 access (The default is block byb 3BB -_-)
        - &#34;[::]:444:443&#34;
      volumes:
        - /.../NPM/data:/data                           # WebUI data
        - /.../NPM/letsencrypt:/etc/letsencrypt         # https certificates

`Offical website &lt;https://nginxproxymanager.com/&gt;`_

After you login to the WebUI, setup a proxy like this:

.. image:: /images/NPM-sample.png
  :width: 400
  :align: center


HTTPS:

.. image:: /images/NPM-SSL.png
  :width: 400
  :align: center


Nextcloud
=========

`Nextcloud &lt;https://nextcloud.com&gt;`_: It is a suite of client-server software for creating and using file hosting services.

.. code-block:: yaml

    db:                                         # database
        image: mariadb:10.6
        restart: always
        command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
        volumes:
          - /.../nextcloud/db:/var/lib/mysql
        environment:
          - MYSQL_ROOT_PASSWORD=CHANGEME
          - MYSQL_PASSWORD=CHANGEME
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud

      redis:
        image: redis:alpine
        restart: always
        networks:
          - redisnet
        expose:
          - 6379

      nextcloud:
        image: nextcloud
        restart: always
        depends_on:
          - db
          - redis
        links:
          - db
        volumes:
          - /.../nextcloud/html:/var/www/html             # all your files
        environment:
          - REDIS_HOST=redis
          - MYSQL_PASSWORD=CHANGEME             # same password as in db
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud
          - MYSQL_HOST=db
          - PHP_MEMORY_LIMIT=1024M
          - PHP_UPLOAD_LIMIT=1024M
        networks:
          redisnet:
          default:
          NPM:
            ipv4_address: 10.5.0.5

`Offical examples &lt;https://github.com/nextcloud/docker/tree/master/.examples&gt;`_

Nextcloud requires some small tuning, check out the auto health-check in setting (click your account icon), and edit ``/.../nextcloud/html/config/config.php`` according to the document.

If Nextcloud is behind NPM, you need to add its address to ``trusted_domains`` and ``trusted_proxies``, for me:

.. code-block:: php

    &#39;trusted_domains&#39; =&gt;
    array (
      0 =&gt; &#39;cloud.bajzc.com&#39;,
      1 =&gt; &#39;10.5.0.5&#39;,
      2 =&gt; &#39;10.5.0.3&#39;,
      3 =&gt; &#39;cloud6.bajzc.com&#39;,
    ),
    &#39;trusted_proxies&#39; =&gt;
    array (
      0 =&gt; &#39;10.5.0.0/16&#39;,
    ),

As NPM is using HTTP to communicate with Nextcloud:

.. code-block:: php

    &#39;overwriteprotocol&#39; =&gt; &#39;https&#39;,
    &#39;overwritecondaddr&#39; =&gt; &#39;^10\\.5\\.0\\.3$&#39;,   // NPM address here
    &#39;forwarded-for-headers&#39; =&gt;
    array (
      0 =&gt; &#39;X-Forwarded-For&#39;,
      1 =&gt; &#39;HTTP_X_FORWARDED_FOR&#39;,
    ),

Docker Compose
==============

All the config scripts I shared are part of the final ``docker-compose.yml`` file.

Install **docker** and `docker-compose &lt;https://docs.docker.com/compose/install/linux/&gt;`_, edit your ``docker-compose.yml`` and run ``docker compose up -d`` under the same folder to start all the applications.

=====================
**Part IV - Backup**
=====================

Considering that you may store lots of personal data, such as photos and documents remotely on this server, you should carefully consider the backup issue.

Tools like snapper could be use to prevent from typing something wrong: ``rm -r /`` instead of ``rm -r ./``. Unfortunately，these tools can not help you save data when a hard disk fault happened.

So, build a RAID array or LVM RAID would be nice, but this require you to have have two or more disks with same size, which are usually more expensive than your server...

Use cold backup, instead, allow you backup with disk with different size. For example, I use an 8TB HDD for the online storage and two 2TB disks for cold backup (offline).


---

> Author: Jason Li  
> URL: https://bajzc.org/posts/home-nas-deployment/  

