---
title: Let's encrypt! (Gentoo)
slug: lets-encrypt-gentoo
date: 2023-11-11T12:29:09+07:00
tags: 
- SSL
- acme
- Gentoo
categories:
- Review
---

.. figure:: /images/https.png
  :align: center


.. note::

  My domain is provided from Google, the method of getting the API token could be different for other provider

.. contents::

Get certificates
================

1. ``emerge -a app-crypt/acme-sh``
2. ``emerge --ask dcron``, acme-sh need this to set cron job to renew the certs automatically
3. Get your domain access token from the provider (`Google Domain <https://domains.google.com/registrar/>`_)
4. For Google Domains:

.. code-block:: bash

  export GOOGLEDOMAINS_ACCESS_TOKEN="<generated-access-token>"

5. run acme-sh as **root**

.. code-block:: bash

  acme.sh --issue --dns dns_googledomains -d bajzc.com -d ipv4.bajzc.com -k ec-384 --ecc

Log:

::

  [Sat Nov 11 13:58:34 CST 2023] Using CA: https://acme.zerossl.com/v2/DV90
  [Sat Nov 11 13:58:34 CST 2023] Creating domain key
  [Sat Nov 11 13:58:34 CST 2023] The domain key is here: /etc/acme-sh//bajzc.com_ecc/bajzc.com.key
  [Sat Nov 11 13:58:34 CST 2023] Multi domain='DNS:bajzc.com,DNS:ipv4.bajzc.com'
  [Sat Nov 11 13:58:34 CST 2023] Getting domain auth token for each domain
  [Sat Nov 11 13:58:42 CST 2023] Getting webroot for domain='bajzc.com'
  [Sat Nov 11 13:58:42 CST 2023] Getting webroot for domain='ipv4.bajzc.com'
  [Sat Nov 11 13:58:43 CST 2023] Adding txt value: eegpTUjIL53Nvy0ES6sS-W3LZA4vVBfQQ for domain:  _acme-challenge.bajzc.com
  [Sat Nov 11 13:58:43 CST 2023] Invoking Google Domains ACME DNS API.
  [Sat Nov 11 13:58:44 CST 2023] Adding TXT record for _acme-challenge.bajzc.com.
  [Sat Nov 11 13:58:47 CST 2023] TXT record added.
  [Sat Nov 11 13:58:47 CST 2023] The txt record is added: Success.
  [Sat Nov 11 13:58:47 CST 2023] Adding txt value: .. for domain:  _acme-challenge.ipv4.bajzc.com
  [Sat Nov 11 13:58:47 CST 2023] Invoking Google Domains ACME DNS API.
  [Sat Nov 11 13:58:50 CST 2023] Adding TXT record for _acme-challenge.ipv4.bajzc.com.
  [Sat Nov 11 13:58:52 CST 2023] TXT record added.
  [Sat Nov 11 13:58:52 CST 2023] The txt record is added: Success.
  [Sat Nov 11 13:58:52 CST 2023] Let's check each DNS record now. Sleep 20 seconds first.
  [Sat Nov 11 13:59:13 CST 2023] You can use '--dnssleep' to disable public dns checks.
  [Sat Nov 11 13:59:13 CST 2023] See: https://github.com/acmesh-official/acme.sh/wiki/dnscheck
  [Sat Nov 11 13:59:14 CST 2023] Checking bajzc.com for _acme-challenge.bajzc.com
  [Sat Nov 11 13:59:14 CST 2023] Domain bajzc.com '_acme-challenge.bajzc.com' success.
  [Sat Nov 11 13:59:14 CST 2023] Checking ipv4.bajzc.com for _acme-challenge.ipv4.bajzc.com
  [Sat Nov 11 13:59:15 CST 2023] Domain ipv4.bajzc.com '_acme-challenge.ipv4.bajzc.com' success.
  [Sat Nov 11 13:59:15 CST 2023] All success, let's return
  [Sat Nov 11 13:59:15 CST 2023] Verifying: bajzc.com
  [Sat Nov 11 13:59:17 CST 2023] Processing, The CA is processing your order, please just wait. (1/30)
  [Sat Nov 11 13:59:22 CST 2023] Success
  [Sat Nov 11 13:59:22 CST 2023] Verifying: ipv4.bajzc.com
  [Sat Nov 11 13:59:24 CST 2023] Processing, The CA is processing your order, please just wait. (1/30)
  [Sat Nov 11 13:59:28 CST 2023] Success
  [Sat Nov 11 13:59:28 CST 2023] Removing DNS records.
  [Sat Nov 11 13:59:28 CST 2023] Removing txt: eegpTUjIL53Nvy0ES6sS-W3LZA4vVBfQQ for domain: _acme-challenge.bajzc.com
  [Sat Nov 11 13:59:28 CST 2023] Invoking Google Domains ACME DNS API.
  [Sat Nov 11 13:59:30 CST 2023] Removing TXT record for _acme-challenge.bajzc.com.
  [Sat Nov 11 13:59:32 CST 2023] TXT record removed.
  [Sat Nov 11 13:59:32 CST 2023] Removed: Success
  [Sat Nov 11 13:59:32 CST 2023] Removing txt: .. for domain: _acme-challenge.ipv4.bajzc.com
  [Sat Nov 11 13:59:32 CST 2023] Invoking Google Domains ACME DNS API.
  [Sat Nov 11 13:59:36 CST 2023] Removing TXT record for _acme-challenge.ipv4.bajzc.com.
  [Sat Nov 11 13:59:38 CST 2023] TXT record removed.
  [Sat Nov 11 13:59:38 CST 2023] Removed: Success
  [Sat Nov 11 13:59:38 CST 2023] Verify finished, start to sign.
  [Sat Nov 11 13:59:38 CST 2023] Lets finalize the order.
  [Sat Nov 11 13:59:38 CST 2023] Le_OrderFinalize='https://acme.zerossl.com/v2/DV90/order/BFZaIPs9kuec-R1zsU1Cqw/finalize'
  [Sat Nov 11 13:59:40 CST 2023] Order status is processing, lets sleep and retry.
  [Sat Nov 11 13:59:40 CST 2023] Retry after: 15
  [Sat Nov 11 13:59:56 CST 2023] Polling order status: https://acme.zerossl.com/v2/DV90/order/BFZaIPs9kuec-R1zsU1Cqw
  [Sat Nov 11 13:59:58 CST 2023] Downloading cert.
  [Sat Nov 11 13:59:58 CST 2023] Le_LinkCert='https://acme.zerossl.com/v2/DV90/cert/gGev2hxpZhQ5UbpPGdiNsA'
  [Sat Nov 11 14:00:00 CST 2023] Cert success.
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
  [Sat Nov 11 14:00:00 CST 2023] Your cert is in: /etc/acme-sh//bajzc.com_ecc/bajzc.com.cer
  [Sat Nov 11 14:00:00 CST 2023] Your cert key is in: /etc/acme-sh//bajzc.com_ecc/bajzc.com.key
  [Sat Nov 11 14:00:00 CST 2023] The intermediate CA cert is in: /etc/acme-sh//bajzc.com_ecc/ca.cer
  [Sat Nov 11 14:00:00 CST 2023] And the full chain certs is there: /etc/acme-sh//bajzc.com_ecc/fullchain.cer

Apache Config
=============

/etc/apache2/vhost.d/ssl.conf:

.. code-block:: cfg

  Listen 443
  <VirtualHost *:443>
    ServerName bajzc.com
    Include /etc/apache2/vhosts.d/default_vhost.include

    SSLCertificateFile /etc/acme-sh/bajzc.com_ecc/bajzc.com.cer
    SSLCertificateKeyFile /etc/acme-sh/bajzc.com_ecc/bajzc.com.key
    SSLCertificateChainFile /etc/acme-sh//bajzc.com_ecc/ca.cer

    ...

  </VirtualHost>

Reference:

1. `Google Docs <https://support.google.com/domains/answer/7630973>`_

2. `Gentoo Docs <https://wiki.gentoo.org/wiki/Let%27s_Encrypt>`_
