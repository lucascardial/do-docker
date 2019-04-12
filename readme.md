---

# Docker Integration

A step by step how to configure a Docker Machine with Swarm on Digital Ocean.

## Droplet and Docker Machine

By first, you need generate your `API TOKEN` from Digital Ocean: https://www.digitalocean.com/docs/api/create-personal-access-token/

So, define the token as ENV VAR in your local machine defining into your prefer terminal profile

```bash
echo "export DO_TOKEN_API=your-token-api" >> ~/.zshrc
source ~/.zshrc
```

### Docker Machine

Install Docker Machine in your computer
Manual: https://docs.docker.com/machine/install-machine/
After installed, create a docker-machine using the digitalocean driver. You can change the "Your-Machine-Name", its will the same name of your DO Droplet.

```bash
docker-machine create --driver digitalocean --digitalocean-access-token $DO_TOKEN_API  Your-Machine-Name
```

This command create a Docker Instance on Digital Ocean Droplet using your DIGITAL OCEAN TOKEN associating this instance (docker machine) in your local docker-machine list.
Now, you can see yours docker-machine running:

```bash
docker-machine ls
```

Pay attention to the URL parameter, you will need it to connect to the droplet later.

### SSH Connection

To connect in your docker-machine (droplet) via ssh, you need first add your ssh key into ~/.ssh/authorized_keys via `docker-machine`:

```bash
# Copy your id_rsa.pub conent to clipboard
pbcopy < ~/.ssh/id_rsa.pub
# Connect in to your droplet
docker-machine ssh Your-Machine-Name
# Paste the clipboard inside quotation marks
echo "CTRL+V" >> ~/.ssh/authorized_keys
exit
```

Now you can access the droplet ssh without docker-machine command:

```bash
ssh root@droplet.ip
```

### Init Swarm

Swarm Documentation: https://docs.docker.com/engine/swarm/

```bash
docker swarm init --advertise-addr <your.droplet.ip>
#example: docker swarm init --advertise-addr 0.0.0.0
```

## Traefik

Documentation: https://docs.traefik.io/

### Creating Service

##### Network

```bash
docker network create --driver=overlay traefik-net
```

###### Volumes

```bash
docker volume create traefik-net-certificates
```

###### Update Docker Node

```bash
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
export docker node update --label-add traefik-net.traefik-net-certificates=true $NODE_ID
```

###### Env vars

```bash
echo "export EMAIL=mail@example.com" >> ~/.bashrc
echo "export USE_HOSTNAME=your-domain.com" >> ~/.bashrc
echo "export USERNAME=admin" >> ~/.bashrc
echo "export PASSWORD=secret" >> ~/.bashrc
echo "export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)" >> ~/.bashrc
```

##### Ports

Expose ports 80 and 443

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Create Traefik Service on Swarm node

```bash
docker service create \
    --name traefik \
    --constraint=node.labels.traefik-net.traefik-net-certificates==true \
    --publish 80:80 \
    --publish 443:443 \
    --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
    --mount type=volume,source=traefik-net-certificates,target=/certificates \
    --network traefik-net \
    --label "traefik.frontend.rule=Host:traefik.$USE_HOSTNAME" \
    --label "traefik.enable=true" \
    --label "traefik.port=8080" \
    --label "traefik.tags=traefik-net" \
    --label "traefik.docker.network=traefik-net" \
    --label "traefik.redirectorservice.frontend.entryPoints=http" \
    --label "traefik.redirectorservice.frontend.redirect.entryPoint=https" \
    --label "traefik.webservice.frontend.entryPoints=https" \
    --label "traefik.frontend.auth.basic.users=${USERNAME}:${HASHED_PASSWORD}" \
    traefik:v1.7 \
    --docker \
    --docker.swarmmode \
    --docker.watch \
    --docker.exposedbydefault=false \
    --constraints=tag==traefik-net \
    --entrypoints='Name:http Address::80' \
    --entrypoints='Name:https Address::443 TLS' \
    --acme \
    --acme.email=$EMAIL \
    --acme.storage=/certificates/acme.json \
    --acme.entryPoint=https \
    --acme.httpChallenge.entryPoint=http\
    --acme.onhostrule=true \
    --acme.acmelogging=true \
    --logLevel=INFO \
    --accessLog \
    --api
```

In your DNS manager, point the subdomain `traefik.your-domain.com` to the `your.droplet.ip`, after DNS propagation try https.traefik.your-domain.com
