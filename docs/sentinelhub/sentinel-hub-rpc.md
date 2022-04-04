# Sentinel Full Node RPC/API
In this document we will describe the process in setting up a full node with RPC access to further decentralise the Sentinel Network. It is mandatory that Sentinel expands access to these types of nodes if it is to become fully decentralised with no single, or small point of failure/censorship. 

## Current RPC/API Public Full Nodes

### Mainnet

| Type    |   Endpoint    |
| ----------- | ---------------- |
|  RPC   |  https://rpc.mathnodes.com:4444   |
|  RPC   |  https://rpc.mathnodes.com:8080   |
|  RPC   |  https://rpc.sentinel.co:443   |
|  RPC   | https://rpc.sentinel.smartnodes.one    |
|  API   | https://api.sentinel.smartnodes.one    |
|  RPC   | https://rpc-sentinel.itastakers.com:443 | 

If you add an RPC or API Node, please make a pull request to this README so we can have a central station for people to locate these services.

## Installtion (Precompiled Binary)
Get the latest binary here:

![https://github.com/sentinel-official/hub/releases/tag/v0.9.2](https://github.com/sentinel-official/hub/releases/tag/v0.9.2)

## Installation (Source)

Requires [Go 1.16+](https://golang.org/dl/)

### Linux

Create a **sentinel** username

```shell
sudo adduser sentinel
sudo -i -u sentinel
```

Grant sudo access to sentinel

```shell
nano /etc/sudoers
```

Add the following line

```
sentinel ALL=(ALL:ALL) ALL
```

Get a copy of **golang** and unpack it

```shell
cd ~ && \
curl -OL https://golang.org/dl/go1.17.linux-amd64.tar.gz && \
tar -C ${HOME} -xvf go1.17.linux-amd64.tar.gz
```

Next edit `.profile` or `.bashrc` and add the following lines to the bottom of the file:

```
export GOPATH=${HOME}/go
export GOBIN=${GOPATH}/bin
export PATH=${PATH}:${GOPATH}/bin:${GOBIN}
```

Source the file
```shell
source .bashrc
```
or 
```shell
source .profile
```

Then set the following variables

```shell
VERSION=v0.9.2
BASE_DIRECTORY=${GOPATH}/src/github.com/sentinel-official
```

Next clone the Sentinel hub source:

```shell
rm -rf ${BASE_DIRECTORY}/hub/ && mkdir -p ${BASE_DIRECTORY} && cd ${BASE_DIRECTORY}/ && \
git clone https://github.com/sentinel-official/hub.git && cd ${BASE_DIRECTORY}/hub/ && \
git checkout ${VERSION}
```

Build and install the software

```shell
make install
```

## Initialize sentinelhub

Initialize the Sentinel Hub config

```shell
MONIKER="Your Node Name Here"
CHAIN_ID="sentinelhub-2"

sentinelhub init ${MONIKER} \
    --chain-id ${CHAIN_ID}
```

Setting your Moniker to your Node's name. 

## Sentinel Blockchain Snapshot

Setup to download latest snapshot of the Sentinel Blockchain

```shell
rm -rf ~/.sentinelhub/data/; \
mkdir -p ~/.sentinelhub/data/; \
cd ~/.sentinelhub/data/
```

Download the latest snapshot

```shell
SNAP_NAME=$(curl -s http://135.181.60.250:8083/sentinel/ | egrep -o ">sentinelhub-2.*tar" | tr -d ">"); \
wget -O - http://135.181.60.250:8083/sentinel/${SNAP_NAME} | tar xf -
```

## SSL Certificates

Create a DNS entry in your nameservice provider to point a domain to your IP address of your Full Node.

Run certbot against your domain with the apache plugin and install the SSL certs

(ensure port *80* is open in your firewall, i.e., `ufw allow 80/tcp`)

```shell
MYDOMAIN="your_domain.com"
```

Changing *your_domain.com* to whatever your have setup.

```shell
sudo apt-get -y install certbot python3-certbot-apache && \
sudo certbot --apache certonly -d ${MYDOMAIN} && \
mkdir ${HOME}/.sentinelhub/certs && \
sudo cp -L /etc/letsencrypt/live/${MYDOMAIN}/fullchain.pem /etc/letsencrypt/live/${MYDOMAIN}/privkey.pem ${HOME}/.sentinelhub/certs && \ 
chown sentinel:sentinel ${HOME}/.sentinelhub/certs/*
```

Now that your certicates have been installed we can configure sentinel hub.

## Sentinel Hub Config

### app.toml

To configure the API access url set the following line in `${HOME}/.sentinelhub/config/app.toml`:

```
address = "tcp://0.0.0.0:1317"
```

Then allow firewall access:

```shell
sudo ufw allow 1317/tcp
```

### config.toml

Edit the following lines
```shell
nano ${HOME}/.sentinelhub/config/config.toml
```

Change the following in their repsective sections

```
moniker = "Your Node Name"

[rpc]
laddr = "tcp://x.x.x.x:4444"
grpc_max_open_connections = 2900
max_open_connections = 2900
max_subscription_clients = 100
timeout_broadcast_tx_commit = 25s
tls_cert_file = "/home/sentinel/.sentinelhub/certs/fullchain.pem"
tls_key_file = "/home/sentinel/.sentinelhub/certs/privkey.pem"

[p2p]
laddr = "tcp://0.0.0.0:26656"
```

Finally set `persistent_peers` in `[p2p]` section equal to the value of 

```shell
curl -fsLS https://raw.githubusercontent.com/sentinel-official/networks/main/sentinelhub-2/persistent_peers.txt
```

Configure your firewall

```shell
sudo ufw allow 4444/tcp && \
sudo ufw allow 26656/tcp
```

**NOTE**: when setting *max_open_connections* greater than the defualt, it requries changing your *soft* file limits.

First verify your soft file limits

```shell
ulimit -Sn
```

if it shows **1024**, you will need to increase this by appending */etc/security/limits.conf* with

```
sentinel       soft    nofile      5000
```

Some systems (Ubuntu) may need to edit the *pam.d* login limits by adding to */etc/pam.d/common-session* the following line

```
session required        pam_limits.so
```

If your limits still are not changing, the catch-all option is to change them globally:

Edit */etc/systemd/system.conf*

```
DefaultLimitNOFILE=65536
```

Reboot the machine

## SystemD Service

Create a systemd service and enable it

```shell
nano /etc/systemd/system/sentinel.service
```

Copy and paste the following
```
[Unit]
Description=SentinelHub
After=network.target

[Service]
ExecStart=/home/sentinel/go/bin/sentinelhub start 
User=sentinel
LimitNOFILE=8192
TimeoutStopSec=30min

[Install]
WantedBy=multi-user.target
```

Enable & Start

```shell
sudo systemctl enable sentinel.service && systemctl start sentinel.service
```

Verify Sentinel Hub has started correctly

```shell
sudo systemctl status sentinel
```

Make sure it shows the status as active. If there are any problems, locate them by reading the logs

```shell
sudo journalctl -xe -u sentinel
```

Verify your rpc port is listening:

```shell
ss -atp | grep "4444"
```

That's it. You are now running a full node.

## Relaying your RPC Node (Optional)

To prevent censorship and become resistent to state actors, it is suggested that you have one or more other VPS or Dedicated Servers that you can relay the traffic from to the full node you just set up. The more the better as this can be a way to dynamically create alternating IP addresses for your Sentinel Node. 

Some states may decide to block access to an IP if it is running an RPC node, as this is a crucial point for the Sentinel Network to operate. 

### Setup

First install *iptables-persistent*

```shell
sudo apt install iptables-persistent
```


On your relay node do the following to enable port forwarding:

```shell
sudo nano /etc/sysctl.conf
```

And add the following

```
net.ipv4.ip_forward=1
```

Update 
```shell
sudo sysctl -p
```

Then begin the firewall rules

```shell
RPCNODE="x.x.x.x"
RELAYNODE="y.y.y.y"
```

where *x.x.x.x* is the IP address of your Sentinel Full RPC Node and *y.y.y.y* is the IP address of your relay node.

Then apply the following iptables rules

```shell
sudo iptables -t nat -A PREROUTING -p tcp --dport 4444 -j DNAT --to-destination ${RPCNODE}:4444 && \
sudo iptables -t nat -A POSTROUTING -p tcp --dport 4444 -d ${RPCNODE} -j SNAT --to-source ${RELAYNODE} && \
sudo iptables -A FORWARD -i ens3 -p tcp --syn --dport 4444 -m conntrack --ctstate NEW -j ACCEPT && \
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

If you are hosting an API Node as well, as configured above, re-enter the above iptables rules using the API port, **1317**, as well. 

On your RPC Node it is more secure to then only allow traffic to port *4444* from your relay node. So delete the rules to allow accecss from all to port *4444* and the API port as well, if you enabled that.

```shell
sudo ufw delete 4444/tcp && \
sudo ufw delete 1317/tcp
```

Then add the rule to only allow traffic from your relay node

```shell
sudo ufw allow from ${RELAYNODE} to ${RPCNODE} port 4444 proto tcp
```

Again, if hosting an API service as well do the same with port *1317*

Save your iptables rules

```shell
sudo sh -c 'iptables-save > /etc/iptables/rules.v4'
```

# Notes

That's it. You have now helped decentralise the Sentinel Network. Pat yourself on the back and hats off to you!


