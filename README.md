# Deprecated

Not going to do any more work here, but will polish [server services](https://github.com/thomasborgen/server-services) instead

> Server services contains adminer and backweb so that you can use one instance of adminer to look at databases runnin on the server and using one backweb instance to backup services.

# Traefik in a docker compose

> This is just a file for me to have a traefik docker compose file that i can copy to my server
> and run docker compose up -d and then have other docker containers register their subdomains to 
> it.

You need to set the env variables shown in the .env.example either by making an `.env` file, exportin variables before running
`docker compose up -d` or just add them in the `docker` command.

when running locally just add the `.env` file, update it and run `docker compose -f docker-compose.local.yml up -d`


## Traefik Deployed on your server

### On the server:

* Create a directory to store your Traefik Docker Compose file:

```bash
mkdir -p /root/code/traefik-public/
```

* Create a docker network that other docker containers will use if they need to be accessed from the outside

```bash
docker network create traefik-public
```

### Locally

* Copy the Traefik Docker Compose file to your server. You could do it by running the command `rsync` in your local terminal:

```bash
rsync -a docker-compose.yml root@your-server.example.com:/root/code/traefik-public/
```

* Copy the env.example file now or you could also edit it locally first if you want.

```bash
cp .env.example .env
```

* Edit vars

* Use openssl to generate the "hashed" version of the password for HTTP Basic Auth:

```bash
openssl passwd -apr1 this-will-be-your-admin-password
```

> edit the .env file with the new password that looks something like: `$apr1$/gcaYTx8$DWaJxuRnx5gU/Y53C3nc8/`

* Change the domain name to your domain name

```bash
DOMAIN=example.com
```

* Create an environment variable with the email for Let's Encrypt, e.g.:

```bash
export EMAIL=admin@example.com
```

**Note**: you need to set a different email, an email `@example.com` won't work.


* Copy the edited .env file to the server by running the command `rsync` in your local terminal:

```bash
rsync -a .env root@your-server.example.com:/root/code/traefik-public/
```

### On the server

### Start the Traefik Docker Compose

Go to the directory where you copied the Traefik Docker Compose file in your remote server:

```bash
cd /root/code/traefik-public/
```

Now with the environment variables set and the `docker-compose.yml` in place, you can start the Traefik Docker Compose running the following command:

```bash
docker compose up -d
```


To register a service with this traefik instance add the following to your services docker compose file:

This requires the env variables DOMAIN and STACK_NAME to be present.

using env_file: required: false, is convenient for local development so you can just have the vars in a file.

For deploying, add them to your github action secrets or whatever you use to deploy.

```yml
networks:
  traefik-public:
    # Allow setting it to false for testing
    external: true

services:

  # My service has no subdomain and registeres to traefik directly to DOMAIN ie test.com
  my-service:
    networks:
      - traefik-public
      - default
    env_file:
      - path: .env
        required: false
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.constraint-label=traefik-public

      - traefik.http.services.${STACK_NAME?Variable not set}.loadbalancer.server.port=8000

      - traefik.http.routers.${STACK_NAME?Variable not set}-http.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-http.entrypoints=http

      - traefik.http.routers.${STACK_NAME?Variable not set}-https.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-https.entrypoints=https
      - traefik.http.routers.${STACK_NAME?Variable not set}-https.tls=true
      - traefik.http.routers.${STACK_NAME?Variable not set}-https.tls.certresolver=le

      # Enable redirection for HTTP and HTTPS
      - traefik.http.routers.${STACK_NAME?Variable not set}-http.middlewares=https-redirect

  # Adminer has a subdomain "adminer". It registeres to adminer.DOMAIN ie: adminer.test.com
  adminer:
    image: adminer
    restart: always
    networks:
      - traefik-public
      - default
    # depends_on:
    #   - db
    environment:
      - ADMINER_DESIGN=pepa-linha-dark
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.constraint-label=traefik-public
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-http.rule=Host(`adminer.${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-http.entrypoints=http
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-http.middlewares=https-redirect
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-https.rule=Host(`adminer.${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-https.entrypoints=https
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-https.tls=true
      - traefik.http.routers.${STACK_NAME?Variable not set}-adminer-https.tls.certresolver=le
      - traefik.http.services.${STACK_NAME?Variable not set}-adminer.loadbalancer.server.port=8080



```
