---
layout: post
title: 为nginx配置Let's Encrypt证书
date: 2019-11-25 10:38:00
author: 任怀林
categories:
- linux
thumb: linux.png
isCJKLanguage: true
---

# 1. 安装[certbot](https://certbot.eff.org)

# 2. Add Certbot PPA

You'll need to add the Certbot PPA to your list of repositories. To do so, run the following commands on the command line on the machine:

```
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
```

# 3. Install Certbot

Run this command on the command line on the machine to install Certbot.

```
$ sudo apt-get install certbot python-certbot-nginx
```

## 4. Choose how you'd like to run Certbot

- Either get and install your certificates...
  
  Run this command to get a certificate and have Certbot edit your Nginx configuration automatically to serve it, turning on HTTPS access in a single step.
  
  ```
  $ sudo certbot --nginx
  Plugins selected: Authenticator nginx, Installer nginx
  Enter email address (used for urgent renewal and security notices) (Enter 'c' to
  cancel): harley@acme.com  #在这里输入一个邮箱，用来提醒证书刷新。
  ```
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  Please read the Terms of Service at
  https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
  agree in order to register with the ACME server at
  https://acme-v02.api.letsencrypt.org/directory
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  (A)gree/(C)ancel: A  # 只能同意
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  Would you be willing to share your email address with the Electronic Frontier
  Foundation, a founding partner of the Let's Encrypt project and the non-profit
  organization that develops Certbot? We'd like to send you email about our work
  encrypting the web, EFF news, campaigns, and ways to support digital freedom.
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  (Y)es/(N)o: N   #不要分享我的邮箱
  
  Which names would you like to activate HTTPS for?
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  1: test.acme.com
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  Select the appropriate numbers separated by commas and/or spaces, or leave input
  blank to select all options shown (Enter 'c' to cancel):
  Obtaining a new certificate
  Performing the following challenges:
  http-01 challenge for wx.jcydjf.com
  Waiting for verification...
  Cleaning up challenges
  Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/test.acme.com
  
  Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  1: No redirect - Make no further changes to the webserver configuration.
  2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
  new sites, or if you're confident your site works on HTTPS. You can undo this
  change by editing your web server's configuration.
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2  #把http的请求转到https上。
  Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/test.acme.com
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  Congratulations! You have successfully enabled https://test.acme.com
  
  You should test your configuration at:
  https://www.ssllabs.com/ssltest/analyze.html?d=test.acme.com
  
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
  
  IMPORTANT NOTES:

- Congratulations! Your certificate and chain have been saved at:
  /etc/letsencrypt/live/test.acme.com/fullchain.pem
  Your key file has been saved at:
  /etc/letsencrypt/live/test.acme.com/privkey.pem
  Your cert will expire on 2020-02-23. To obtain a new or tweaked
  version of this certificate in the future, simply run certbot again
  with the "certonly" option. To non-interactively renew *all* of
  your certificates, run "certbot renew"

- Your account credentials have been saved in your Certbot
  configuration directory at /etc/letsencrypt. You should make a
  secure backup of this folder now. This configuration directory will
  also contain certificates and private keys obtained by Certbot so
  making regular backups of this folder is ideal.

- If you like Certbot, please consider supporting our work by:
  
  Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
  Donating to EFF:                    https://eff.org/donate-le

- Or, just get a certificate
  
  If you're feeling more conservative and would like to make the changes to your Nginx configuration by hand, run this command.

```
  $ sudo certbot certonly --nginx
```

## 5. Test automatic renewal

The Certbot packages on your system come with a cron job or systemd timer that will renew your certificates automatically before they expire. You will not need to run Certbot again, unless you change your configuration. You can test automatic renewal for your certificates by running this command:

```
$ sudo certbot renew --dry-run
```

The command to renew certbot is installed in one of the following locations:

- /etc/crontab/
- /etc/cron.*/*
- systemctl list-timers
