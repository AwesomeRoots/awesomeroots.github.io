---
published: true
title: Enabling Single Sign On for daemons with PAM and Jasig CAS
layout: post
tags: [c, devops, cas, sso]
---
Jasig CAS is a great service for single sign on. Basically it is a Java application that, once deployed, provides a single authentication mechanism for all your web services. Thus, if you're developing a multicomponent portal, which consists of several sites — CAS is a great solution.

With CAS, users can authenticate only once and get access to all sites. That's absolutely the same thing that Google does — you sign into your GMail account, and get the access to YouTube, Blogspot, Calendar, Play Market and other awesome services hosted on different domains.

That's completely different from what OAuth and OpenID do. Last two provide user's credentials upon authorization request. Your app still has to manage user accounts, it just gets another way of verifying credentials. With CAS, it all will be done behind the scenes. User just comes to the site, gets transparently redirected several times, and he's already in.

The only problem is — CAS provides only web based auth. But what if you want to authenticate a daemon? That's also possible — let's take Dovecot as an example. We will use [PAM](https://en.wikipedia.org/wiki/Pluggable_authentication_module) authentication mechanism. Several modules for doing that are [suggested](https://wiki.jasig.org/display/casc/pam+module) on official site, but the only problem with those is — they don't work.

That's why our developer had to pick the most recent [module](https://github.com/atiti/pam_cas-reloaded), fork it and update for recent CAS versions. [This version](https://github.com/isanosyan/pam_cas-reloaded/tree/ubuntu_12.04_64bit+cas_3.5.1) works fine on 64 bit Ubuntu 12.04 LTS with CAS 3.5.1.

And here's the configuration for PAM (`/etc/pam.d/dovecot` in our case):

```plain
#%PAM-1.0

auth       sufficient   pam_unix.so nullok_secure
auth       sufficient   pam_cas.so use_first_pass
auth       required     pam_deny.so

account    sufficient   pam_cas.so
account    required     pam_deny.so
```

And `pam_cas` config (`/etc/pam_cas.conf`):

```plain
[General]

; This is the target service URL
; Could be a website to redirect to, or a service such as
; ssh/imap/etc
; SERVICE_URL = ssh
SERVICE_URL = imap://my.dovecot.installation/

; This is the callback for a proxy ticket
SERVICE_CALLBACK_URL = http://my.dovecot.installation/?_action=pgtcallback

; CAS BASE URL
; (No need for trailing slash)
CAS_BASE_URL = https://my.cas.installation/cas

; Enable serviceTicket validation
;
; This option is there to allow disabling/enabling of user+serviceTicket logins
ENABLE_ST = 0


; Enable proxyTicket validation
;
; Same as above, except for proxy tickets
ENABLE_PT = 1

; Enable user+pass validation
;
; Enable user+pass login against CAS
ENABLE_UP = 0
```

After enabling the module, once user has logged into any of your web sites, he gets an access to his mailbox in your system without additional prompt. Hope you find it useful!
