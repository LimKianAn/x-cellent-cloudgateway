# X-cellent Cloudgateway

At [x-cellent](https://www.x-cellent.com/), we are building and maintaining [cloud-native systems](https://metal-stack.io/) and often we are faced with complicated networking problems when we try to connect two applications built with different generations of technologies. On the one hand, the network traffic is often, if not always, regulated by multiple firewalls and/or NAT. On the other hand, some legacy systems use private IP addresses outside of the RFC 1918 space, making direct routing impossible. Here comes **X-cellent Cloudgateway** to the rescue.

## Let's start small

The following is a [minimal](https://github.com/LimKianAn/x-cellent-cloudgateway/demo/minimal/docker-compose.yaml) [docker compose](https://docs.docker.com/compose/install/) file which shows the power of cloudgateway.

```yaml
version: "3.8"
networks:
  legacy:
    name: legacy
  cloudnative:
    name: cloudnative
services:
  nginx:
    networks:
      - legacy
    image: nginx
  server:
    networks:
      - legacy
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./server:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
  client:
    networks:
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
    ports:
      - 8080:8080
```

We have two docker networks: `legacy` and `cloudnative`, which represent two private networks. There's an nginx process running in the former network and we want to tap into that from the latter network. How can we go about achieving that? We build a tunnel by running two cloudgateway processes in these two networks respectively. Let's see that in action!

```sh
git clone git@github.com:LimKianAn/x-cellent-cloudgateway.git
cd demo/minimal
docker compose up
```

Now, we have our minimal demo running. Let's open another terminal and run `curl localhost:8080`. Voilà! We should see the default welcome page of nginx.
