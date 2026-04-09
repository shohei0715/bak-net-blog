---
title: "CPIにownCloud 8をインストールする"
date: 2015-06-10 21:18:52
draft: false
slug: "cpiにowncloud-8をインストールする"
---

htaccessをこうすれば動く

[text]
# Version: 8.0.2

Options +FollowSymLinks
 
 
 
 SetEnvIfNoCase ^Authorization$ "(.+)" XAUTHORIZATION=$1
 RequestHeader set XAuthorization %{XAUTHORIZATION}e env=XAUTHORIZATION
 
 
 
 
 php_value upload_max_filesize 513M
 php_value post_max_size 513M
 php_value memory_limit 512M
 php_value mbstring.func_overload 0
 php_value always_populate_raw_post_data -1
 
 SetEnv htaccessWorking true
 
 
 
 RewriteEngine on
 RewriteRule .* - [env=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
 RewriteRule ^\.well-known/host-meta /public.php?service=host-meta [QSA,L]
 RewriteRule ^\.well-known/host-meta\.json /public.php?service=host-meta-json [QSA,L]
 RewriteRule ^\.well-known/carddav /remote.php/carddav/ [R]
 RewriteRule ^\.well-known/caldav /remote.php/caldav/ [R]
 RewriteRule ^apps/calendar/caldav\.php remote.php/caldav/ [QSA,L]
 RewriteRule ^apps/contacts/carddav\.php remote.php/carddav/ [QSA,L]
 RewriteRule ^remote/(.*) remote.php [QSA,L]
 RewriteRule ^(build|tests|config|lib|3rdparty|templates)/.* /503.php [L]
 RewriteRule ^(\.|autotest|occ|issue|indie|db_|console).* /503.php [L]
 #RewriteRule ^(build|tests|config|lib|3rdparty|templates)/.* - [R=404,L]
 #RewriteRule ^(\.|autotest|occ|issue|indie|db_|console).* - [R=404,L]
 
 
 AddType image/svg+xml svg svgz
 AddEncoding gzip svgz
 
 
 DirectoryIndex index.php index.html
 
 AddDefaultCharset utf-8
 Options -Indexes
 
 ModPagespeed Off
 
 
 
 Header set Cache-Control "max-age=7200, public"
 
 

ErrorDocument 403 /owncloud/core/templates/403.php
 ErrorDocument 404 /owncloud/core/templates/404.php

 Order allow,deny
 Allow from all
 
 
 Order deny,allow
 Deny from all
 
 
 Dav Off
 
[/text]
