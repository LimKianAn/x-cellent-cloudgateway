# X-cellent Cloudgateway

At [x-cellent](https://www.x-cellent.com/), we are building and maintaining [cloud-native systems](https://metal-stack.io/) and often we are faced with complicated networking problems when we try to connect two applications built with different generations of technologies. On the one hand, the network traffic is often, if not always, regulated by multiple firewalls and/or NAT. On the other hand, some legacy systems use private IP addresses outside of the RFC 1918 space, making direct routing impossible. Here comes cloudgateway to the rescue.

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

Before we `docker exec` into the containers we just created, let's have a look at the logs of `docker compose up`. Two specific lines are of interest to us. (They might not be adjacent to each other.)

```termial
minimal-server-1  | [#] ip -4 address add 10.192.0.121/24 dev wg0
...
minimal-client-1  | [#] ip -4 address add 10.192.0.245/32 dev wg0
```

The newly created interface `wg0` in each container is assigned an IP address and a CIDR mask. The IP address is clear, meaning the IP address in the VPN. The CIDR masks of the server and the client are different. For the server `/24` implies that there are peers assigned IP addresses in the range `10.192.0.0/24`, which means multiple clients. In contrast, for the client `/32` doesn't imply a range of peers since it's only going to talk to the server.

## VPN with WireGuard&reg;

Under the hood, cloudgateway is powered by [WireGuard&reg;](https://www.wireguard.com/). Let's see that in action! In one terminal, run:

```sh
docker exec -it minimal-server-1 bash
```

In another terminal, run:

```sh
docker exec -it minimal-client-1 bash
```

In each terminal, run:

```sh
wg
```

The results in the server terminal would read:

```terminal
interface: wg0
  public key: 2o3hItYcvPrcmDMog6rOhmdzZd6PH+QIZtCvZnVrslU=
  private key: (hidden)
  listening port: 8765

peer: QqdKxpGpG1TTuyjCayMpRRKwLm5kjyIKZybxjv8oDnQ=
  endpoint: 172.24.0.2:55748
  allowed ips: 10.192.0.245/32
  latest handshake: 2 minutes, 34 seconds ago
  transfer: 948 B received, 860 B sent
```

The results in the client terminal would read:

```terminal
interface: wg0
  public key: QqdKxpGpG1TTuyjCayMpRRKwLm5kjyIKZybxjv8oDnQ=
  private key: (hidden)
  listening port: 55748

peer: 2o3hItYcvPrcmDMog6rOhmdzZd6PH+QIZtCvZnVrslU=
  endpoint: 172.24.0.3:8765
  allowed ips: 10.192.0.121/32
  latest handshake: 2 minutes, 47 seconds ago
  transfer: 860 B received, 1.07 KiB sent
```

Notice that for `wg0` in the server container, the `listening port` is `8765`, which is assigned by cloudgateway. In contrast, for `wg0` in the client container, the `listening port` is randomly assigned by WireGuard. The reason behind this is that cloudgateway was conceived where we need a server sitting in a legacy-network and this server should respond to requests from applications in a cloud-native environment, like kubernetes. In order to let other clients find the server in the public, the port of the server needs to be fixed (?).

The `endpoint` of a peer is an endpoint in the public, so that the server and client can find each. In our docker-compose example, the endpoint is the IP address of the container. We can prove that in another terminal.

```sh
docker inspect minimal-server-1 -f "{{ .NetworkSettings.Networks.cloudnative.IPAddress }}"
docker inspect minimal-client-1 -f "{{ .NetworkSettings.Networks.cloudnative.IPAddress }}"
```

Let's ping the server's VPN IP address from the client!

```sh
# at client
ping 10.192.0.121
```

Let's see what we've got in the server!

```sh
# at server
tcpdump --interface wg0
```

The results should read:

```terminal
listening on wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
13:02:16.129213 IP 10.192.0.245 > 10.192.0.121: ICMP echo request, id 64908, seq 1, length 64
13:02:16.129257 IP 10.192.0.121 > 10.192.0.245: ICMP echo reply, id 64908, seq 1, length 64
...
```

We should see ICMP echo requests and replies being exchanged between the server and the client through the interface wg0. Note that `wg0` is a virtual interface constructed by VPN protocol, in our case WireGuard&reg;. There must be another real interface involved. Let's see that in action!

Interrupt the running `tcpdump` command at the server and run it again with the interface eth0.

```sh
# at server
tcpdump --interface eth0
```

The results should read:

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:03:56.499668 IP minimal-client-1.cloudnative.51764 > e3b9d54cbacf.8765: UDP, length 128
14:03:56.499952 IP e3b9d54cbacf.8765 > minimal-client-1.cloudnative.51764: UDP, length 128
...
```

This time, we should see UDP packets being exchanged between the client and the server. `minimal-client-1.cloudnative.51764` represents the IP address and the UDP port of the client container, `e3b9d54cbacf.8765` the server's. We can examine that with the tool `nslookup`.

```sh
apt update && apt install nsutils -y
nslookup minimal-client-1
nslookup e3b9d54cbacf
```

We shall see the IP addresses of the containers are listed.

In the previous section, we examined the traffic between VPN IP addresses at wg0 interface, which are backed by the
underlying UDP communication at normal eth0 interface. How about if we ping the server's public IP address directly? Will we see anything at server's wg0 interface?

```sh
# at client
ping 172.24.0.2 # server's IP address
```

If we just run the same command as last time `tcpdump --interface wg0` at the server, we would get nothing. Why? Remember that only the traffic between VPN IP addresses, in our case 10.192.0.121 at the server and 10.192.0.245 at the client, is going through the wg0 interfaces on the both sides. Apart from those, it's just business as usual and the public interface eth0 takes charge.

```sh
# at server
tcpdump --interface eth0
```

We shall observe ICMP echos between the server and the client containers as follows.

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:54:57.778659 IP minimal-client-1.cloudnative > e3b9d54cbacf: ICMP echo request, id 63736, seq 11, length 64
14:54:57.778687 IP e3b9d54cbacf > minimal-client-1.cloudnative: ICMP echo reply, id 63736, seq 11, length 64
```

## Pipe

Another pillar of cloudgateway is _Pipe_, which can be found in cloudgateway [conf.yaml](https://github.com/LimKianAn/x-cellent-cloudgateway/demo/minimal/client/conf.yaml) which is consumed by the container as shown in the docker-compose.yaml above.

```yaml
name: nginx
port: 8080
remote: nginx:80
```

- The `name` is just a name for humans.
- The `port` is the listening port of the build-in TCP proxy of cloudgateway. Each pipe has it's own port.
- The `remote` part is the endpoint of a remote service of interest.

Let's examine this port a bit. Run the following command inside the server and client container:

```sh
ss -tulpen
```

The outputs from the server should read as follows:

```terminal
Netid       State        Recv-Q        Send-Q               Local Address:Port                Peer Address:Port       Process
udp         UNCONN       0             0                       127.0.0.11:56608                    0.0.0.0:*           ino:27370005 sk:15 cgroup:unreachable:e8de1 <->
udp         UNCONN       0             0                          0.0.0.0:8765                     0.0.0.0:*           ino:27372057 sk:16 cgroup:unreachable:1 <->
udp         UNCONN       0             0                             [::]:8765                        [::]:*           ino:27372058 sk:17 cgroup:unreachable:1 v6only:1 <->
tcp         LISTEN       0             4096                    127.0.0.11:46841                    0.0.0.0:*           ino:27370006 sk:18 cgroup:unreachable:e8de1 <->
tcp         LISTEN       0             4096                             *:9000                           *:*           users:(("cloudgateway",pid=1,fd=11)) ino:27359215 sk:19 cgroup:unreachable:1 v6only:0 <->
tcp         LISTEN       0             4096                             *:8080                           *:*           users:(("cloudgateway",pid=1,fd=10)) ino:27359212 sk:1a cgroup:unreachable:1 v6only:0 <->
```

- Two ports bound to `127.0.0.11` are docker DNS handlers. They appear when we specify our own docker network.
- The port `8765` bound to all addresses is for UDP communication in VPN as mentioned in the last section.
- The port `9000` is for the built-in metrics server.
- The port `8080` is the listening port of the build-in TCP proxy for the pipe above.

The outputs from the client is basically the same, except the random port for UDP communication in VPN.

The `remote` part of the pipe, `nginx:80` in our case, is more intricate. At the server, it's left intact since `nginx` is a known domain name in the _legacy_ network. Remember that the server and the nginx containers sit in the same network. At the client, it's translated to `{server's VPN IP}:{proxy port}`, `10.192.0.121:8080` in our case. Let's see that in action.

```sh
# at server
tcpdump --interface wg0 -n
```

```sh
# at client
curl localhost:8080
```

We shall see the welcome page of nginx at the client and a couple of packets exchanged by the client and the server as follows.

```terminal
listening on wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
15:44:43.961628 IP 10.192.0.245.57990 > 10.192.0.121.8080: Flags [S], seq 1160242459, win 64860, options [mss 1380,sackOK,TS val 331828805 ecr 0,nop,wscale 7], length 0
15:44:43.961682 IP 10.192.0.121.8080 > 10.192.0.245.57990: Flags [S.], seq 3478160883, ack 1160242460, win 64296, options [mss 1380,sackOK,TS val 1567599720 ecr 331828805,nop,wscale 7], length 0
...
```

We recognize the translated endpoint of the remote service, `10.192.0.121.8080` in our case, but what about the source port, `57990`? We don't see this port if we run `ss -tulpen` at the client. Let's dive in!

All the traffic arriving at the pipe's port is handled by the TCP proxy, where there're two TCP connections constantly being synced. One is called source and the other one is called destination. The source connection connects to the listening port, _8080_ is our case and the destination connection connects to the endpoint of the remote service. At client, these two connections look as follows:

```terminal
Source: 127.0.0.1:8080 (local) <-> 127.0.0.1:53236 (remote)
Destination: 10.192.0.245:57990 (local) <-> 10.192.0.121:8080 (remote)
```

Similarly at the server:

```terminal
Source: 10.192.0.121:8080 (local) <-> 10.192.0.245:57990 (remote)
Destination: 172.25.0.2:40160 (local) <-> 172.25.0.3:80 (remote)
```

Note that the destination TCP connection at the client is exactly the source one at the server with the reverse local and remote endpoints. The port _57990_ we observed in the outputs of `tcpdump` is exactly the one involved in this very TCP connection.

Let us finish this section by examining the traffic at the interface `eth0` again.

```sh
# at server
tcpdump --interface eth0 -n
```

```sh
# at client
curl localhost:8080
```

At the server, we shall see the similar outputs as follows.

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:46:58.961805 IP 172.24.0.2.55142 > 172.24.0.3.8765: UDP, length 96
08:46:58.961846 IP 172.24.0.3.8765 > 172.24.0.2.55142: UDP, length 96
...
```

As we expected, the traffic between the server and the client goes through the VPN's UDP ports.

The life of a packet would look as follows when we run `curl localhost:8080` at the host.

```terminal
localhost:8080 (mapped to the _client_ container port 8080, the configured port of a pipe)
-> client-container-IP:8080
-> cloudgateway's TCP proxy syncing source and destination TCP connections
-> server-VPN-IP:8080 (traffic actually going through VPN UDP ports)
-> cloudgateway's TCP proxy syncing two connections again
-> nginx-container-IP:80 (nginx port)
```

## Reverse Pipe

How about if we want to reach a service in the cloud-native network from a client in the legacy network? Reverse pipe comes to rescue. In contrast to normal pipes, where a service is resolvable within the server's network without going through the VPN to another network, a revere pipe is for the scenario where a service does sit in another network and the server needs the VPN to fetch it. Here's the [docker-compose.yaml](https://github.com/LimKianAn/x-cellent-cloudgateway/demo/reverse/docker-compose.yaml) for such a reverse pipe.

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
      - cloudnative
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
  client-cloudnative:
    networks:
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client/cloudnative:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
  client-legacy:
    networks:
      - legacy
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client/legacy:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
    ports:
      - 8080:8080
```

This time, nginx is sitting in the cloud-native network instead of the legacy one and there are two clients instead of one, each sitting in their respective network. Note that the port _8080_ is mapped the port _8080_ in the _client-legacy_ container to mimic the case. Let's see the reverse pipe in action.

```sh
# in the project root
cd demo/reverse
docker compose up
```

Then, open another terminal and run `curl localhost:8080`. Voilà! We shall see the default welcome page of nginx again.

Let's go through the life of a packet.

```terminal
localhost:8080 (mapped to the client-legacy container port 8080)
-> client-legacy-container-IP:8080
-> cloudgateway's TCP proxy syncing two TCP connections
-> server-VPN-IP:8080 (traffic actually going through VPN UDP ports)
-> cloudgateway's TCP proxy syncing again
-> client-cloudnative-VPN-IP:8080 (VPN UDP communications)
-> cloudgateway's TCP proxy syncing again
-> nginx-container-IP:80
```

Finally, let's check the config files of cloudgateway in each container. Note that for the server and the client in the cloudnative network, we have to specify the peer:

```yaml
name: nginx
port: 8080
remote: nginx:80
peer: client-cloudnative
```

However, for the client in the legacy network it's the same as the previous demo:

```yaml
name: nginx
port: 8080
remote: nginx:80
```

Why is that? Actually, in the eyes of the client in the legacy network, nothing has changed. It's still a pipe which must go through the server. Therefore, nothing needs to change. For the server and the client in the cloudnative network, it's another story. We have to tell cloudgateway where the service, nginx in our case, is resolvable. To put it another way, we have to tell cloudgateway to which peer the traffic has to be sent. Therefore, the name of this peer has to match the one listed under `peers` in the server's config file. To cut it short: specify the peer of a pipe for the server and the client where the remote endpoint of the service is resolvable.
