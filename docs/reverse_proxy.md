# Reverse-Proxy Configuration

## Gotify with Common-Proxies

All of the following examples will have Gotify hosted at `127.0.0.1:8000` and Common-Proxies hosted at `127.0.0.1:5000`. You can adjust the ports according to your needs.

### Nginx

```nginx
server {

	listen 0.0.0.0:80;
	listen [::1]:80 ;
	
	listen 0.0.0.0:443 ssl;
	listen [::1]:443 ssl;

	# Here goes your domain / subdomain
	server_name gotify.example.com;

	# this sends traffic to common-proxies
	location ~ ^/(FCM|UP|_matrix) {	
		proxy_pass			http://127.0.0.1:5000;
	}

	# this controls gotify traffic
	location / {
		# We set up the reverse proxy
		proxy_pass		http://gotify;
		proxy_http_version		1.1;
	
		# Ensuring it can use websockets
		proxy_set_header	Upgrade $http_upgrade;
		proxy_set_header	Connection "upgrade";
		proxy_set_header	X-Real-IP $remote_addr;
		proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header	X-Forwarded-Proto http;
		proxy_redirect		http:// $scheme://;
	
		# The proxy must preserve the host because gotify verifies the host with the origin
		# for WebSocket connections
		proxy_set_header	Host $http_host;
	
		# These sets the timeout so that the websocket can stay alive
		proxy_connect_timeout	1m;
		proxy_send_timeout		1m;
		proxy_read_timeout		1m;
	}
}
```

### Caddy

This snippet can be placed in a Caddyfile.
```caddy
gotify.example.org {
    @rewrite_proxy {
        path /UP* /_matrix*
    }
    reverse_proxy @rewrite_proxy 127.0.0.1:5000

    reverse_proxy 127.0.0.1:8000
}
```

### Apache

```apache
ServerName gotify.example.org

# Send these 3 paths to common-proxies
ProxyPass "/_matrix" http://127.0.0.1:5000/_matrix
ProxyPass "/UP" http://127.0.0.1:5000/UP
ProxyPass "/FCM" http://127.0.0.1:5000/FCM

Keepalive On

# The proxy must preserve the host because gotify verifies the host with the origin
# for WebSocket connections
ProxyPreserveHost On

# Proxy web socket requests to /stream
ProxyPass "/stream" ws://127.0.0.1:8000/stream retry=0 timeout=60

# Proxy all other requests to /
ProxyPass "/" http://127.0.0.1:8000/ retry=0 timeout=5

ProxyPassReverse / http://127.0.0.1:8000/
```
