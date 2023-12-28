# NGINX reverse proxy 
## HTTP only
### Basic docker compose for nginx

```sh
version: '3'

services:
  webserver:
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html
      - ./logs:/var/log/nginx
    restart: always
```

### Config file for http only

- Create `conf` file inside `nginx/conf` folder.

- If we only want to use http, create file `${site}.conf` (for example `sisued.vn`).

```sh
server {
    listen 80;
    server_name ${site} www.${site};

    location / {
        proxy_pass http://${internal-host}:${port};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Now, nginx route request from port `80` to `${internal-host}:${port}`.

## HTTPS with certbot

### NGINX as webserver

- We should start a basic nginx container to init required `certbot` folders first.

```sh
version: '3'

services:
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    restart: always
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro
      - ./certbot/www:/var/www/certbot/:ro
```

- Add the following configuration file into our `./nginx/conf/` local folder.

```sh
server {
    listen 80;
    listen [::]:80;

    server_name example.org www.example.org;
    server_tokens off;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://example.org$request_uri;
    }
}
```

The configuration is simple. We explain to nginx that it has to listen to port `80` (either on IPv4 or IPv6) for the specific domain name example.org.

By default, we want to redirect someone coming on port `80` to the same route but on port `443`. That's what we do with the `location /` block.

But the specificity here is the other location block. It serves the files Certbot need to authenticate our server and to create the HTTPS certificate for it.

Basically, we say "always redirect to HTTPS except for the `/.well-know/acme-challenge/` route".

We can now reload nginx by doing a rough `docker compose restart` or if you want to avoid service interruptions (even for a couple of seconds) reload it inside the container using `docker compose exec webserver nginx -s reload`.

### Create the certificate using Certbot

For now, nothing will be shown because nginx keeps redirecting you to a 443 port that's not handled by nginx yet. But everything is fine. We only want Certbot to be able to authenticate our server.

To do so, we need to use the docker image for certbot and add it as a service to our Docker Compose project.

```sh
version: '3'

services:
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    restart: always
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro
      - ./certbot/www:/var/www/certbot/:ro
  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
```

We now have two services, one for nginx and one for Certbot. You might have noticed they have declared the same volume. It is meant to make them communicate together.

Certbot will write its files into `./certbot/www/` and nginx will serve them on port `80` to every user asking for `/.well-know/acme-challenge/`. That's how Certbot can authenticate our server.

Note that for Certbot we used `:rw` which stands for "read and write" at the end of the volume declaration. If you don't, it won't be able to write into the folder and authentication will fail.

You can now test that everything is working by running `docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d example.org`. You should get a success message like "The dry run was successful".

Now that we can create certificates for the server, we want to use them in nginx to handle secure connections with end users' browsers.

Certbot create the certificates in the `/etc/letsencrypt/` folder. Same principle as for the webroot, we'll use volumes to share the files between containers.

```sh
version: '3'

services:
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    restart: always
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro
      - ./certbot/www:/var/www/certbot/:ro
      - ./certbot/conf/:/etc/nginx/ssl/:ro
  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
      - ./certbot/conf/:/etc/letsencrypt/:rw
```

Restart your container using `docker compose restart`. Nginx should now have access to the folder where Certbot creates certificates.

However, this folder is empty right now. Re-run Certbot without the `--dry-run` flag to fill the folder with certificates:

```sh 
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.org
```

Since we have those certificates, the piece left is the 443 configuration on nginx.

```sh
server {
    listen 80;
    listen [::]:80;

    server_name example.org www.example.org;
    server_tokens off;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://example.org$request_uri;
    }
}

server {
    listen 443 default_server ssl http2;
    listen [::]:443 ssl http2;

    server_name example.org;

    ssl_certificate /etc/nginx/ssl/live/example.org/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/example.org/privkey.pem;
    
    location / {
    	# ...
    }
}
```

### Renewing the certificates

One small issue you can have with Certbot and Let's Encrypt is that the certificates last only 3 months. You will regularly need to renew the certificates you use if you don't want people to get blocked by an ugly and scary message on their browser.

But since we have this Docker environment in place, it is easier than ever to renew the Let's Encrypt certificates!

```sh
docker compose run --rm certbot renew
```