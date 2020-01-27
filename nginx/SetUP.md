# Backup Current Config 

```
#Backup current configuration:
tar -czvf nginx_$(date +'%F_%H-%M-%S').tar.gz nginx.conf sites-available/ sites-enabled/ nginxconfig.io/

#Unzip the uploaded archive:
unzip -o nginxconfig.io-OpenBSD-test.zip
```

# SSL init

```
#Create a common ACME-challenge directory (for Let's Encrypt):
mkdir -p /var/www/_letsencrypt
chown www-data /var/www/_letsencrypt
```


# CertBOT

```
#Comment out SSL related directives in configuration:
sed -i -r 's/(listen .*443)/\1;#/g; s/(ssl_(certificate|certificate_key|trusted_certificate) )/#;#\1/g' /etc/nginx/sites-available/OpenBSD-test.conf

#Reload NGINX:
sudo nginx -t && sudo systemctl reload nginx

#Obtain certificate:
certbot certonly --webroot -d "OpenBSD-test" -d "www.OpenBSD-test" --email info@OpenBSD-test -w /var/www/_letsencrypt -n --agree-tos --force-renewal

#Uncomment SSL related directives in configuration:
sed -i -r 's/#?;#//g' /etc/nginx/sites-available/OpenBSD-test.conf

#Reload NGINX:
sudo nginx -t && sudo systemctl reload nginx

#Configure Certbot to reload NGINX after success renew:
echo -e '#!/bin/bash\nnginx -t && systemctl reload nginx' | sudo tee /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh
sudo chmod a+x /etc/letsencrypt/renewal-hooks/post/nginx-reload.sh

```

# /etc/nginx/nginx.conf

```
user www-data;
pid /run/nginx.pid;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
	multi_accept on;
	worker_connections 65535;
}

http {
	charset utf-8;
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	types_hash_max_size 2048;
	client_max_body_size 16M;

	# MIME
	include mime.types;
	default_type application/octet-stream;

	# logging
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log warn;

	# SSL
	ssl_session_timeout 1d;
	ssl_session_cache shared:SSL:10m;
	ssl_session_tickets off;

	# Mozilla Modern configuration
	ssl_protocols TLSv1.3;

	# OCSP Stapling
	ssl_stapling on;
	ssl_stapling_verify on;
	resolver 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 208.67.222.222 208.67.220.220 valid=60s;
	resolver_timeout 2s;

	# load configs
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}
```

# /etc/nginx/sites-available/OpenBSD-test.conf

```
server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;

	server_name OpenBSD-test;
	set $base /var/www/OpenBSD-test;
	root $base/public;

	# SSL
	ssl_certificate /etc/letsencrypt/live/OpenBSD-test/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/OpenBSD-test/privkey.pem;
	ssl_trusted_certificate /etc/letsencrypt/live/OpenBSD-test/chain.pem;

	# security
	include nginxconfig.io/security.conf;

	# logging
	access_log /var/log/nginx/OpenBSD-test.access.log;
	error_log /var/log/nginx/OpenBSD-test.error.log warn;

	# index.html fallback
	location / {
		try_files $uri $uri/ /index.html;
	}

	# Python
	location / {
		include nginxconfig.io/python_uwsgi.conf;
	}

	# Django media
	location /media/ {
		alias $base/media/;
	}

	# Django static
	location /static/ {
		alias $base/static/;
	}

	# additional config
	include nginxconfig.io/general.conf;
}

# subdomains redirect
server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;

	server_name *.OpenBSD-test;

	# SSL
	ssl_certificate /etc/letsencrypt/live/OpenBSD-test/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/OpenBSD-test/privkey.pem;
	ssl_trusted_certificate /etc/letsencrypt/live/OpenBSD-test/chain.pem;

	return 301 https://OpenBSD-test$request_uri;
}

# HTTP redirect
server {
	listen 80;
	listen [::]:80;

	server_name .OpenBSD-test;

	include nginxconfig.io/letsencrypt.conf;

	location / {
		return 301 https://OpenBSD-test$request_uri;
	}
}
```

# /etc/nginx/nginxconfig.io/letsencrypt.conf

```
# ACME-challenge
location ^~ /.well-known/acme-challenge/ {
	root /var/www/_letsencrypt;
}
```

# /etc/nginx/nginxconfig.io/security.conf

```
# security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# . files
location ~ /\.(?!well-known) {
	deny all;
}
```

# /etc/nginx/nginxconfig.io/general.conf

```
# favicon.ico
location = /favicon.ico {
	log_not_found off;
	access_log off;
}

# robots.txt
location = /robots.txt {
	log_not_found off;
	access_log off;
}

# assets, media
location ~* \.(?:css(\.map)?|js(\.map)?|jpe?g|png|gif|ico|cur|heic|webp|tiff?|mp3|m4a|aac|ogg|midi?|wav|mp4|mov|webm|mpe?g|avi|ogv|flv|wmv)$ {
	expires 7d;
	access_log off;
}

# svg, fonts
location ~* \.(?:svgz?|ttf|ttc|otf|eot|woff2?)$ {
	add_header Access-Control-Allow-Origin "*";
	expires 7d;
	access_log off;
}

# gzip
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
```

# /etc/nginx/nginxconfig.io/python_uwsgi.conf

```
# default uwsgi_params
include uwsgi_params;

# uwsgi settings
uwsgi_pass						unix:/tmp/uwsgi.sock;
uwsgi_param Host				$host;
uwsgi_param X-Real-IP			$remote_addr;
uwsgi_param X-Forwarded-For		$proxy_add_x_forwarded_for;
uwsgi_param X-Forwarded-Proto	$http_x_forwarded_proto;
```

