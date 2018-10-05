# Here are the exact steps I did to install cyphernode on a debian server running on x86 arch, as user debian.

## Update server and install git

```shell
sudo apt-get update ; sudo apt-get upgrade ; sudo apt-get install git
```

## Docker installation: https://docs.docker.com/install/linux/docker-ce/debian/

```shell
sudo apt-get install      apt-transport-https      ca-certificates      curl      gnupg2      software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/debian \
$(lsb_release -cs) \
stable"
sudo apt-get update
sudo apt-get install docker-ce
sudo groupadd docker
sudo usermod -aG docker $USER
```

CTRL-D (re-login)

## Cyphernode configuration

```shell
docker swarm init --task-history-limit 1
docker network create --driver=overlay --attachable --opt encrypted cyphernodenet
git clone https://github.com/SatoshiPortal/cyphernode.git
cd cyphernode/
vi proxy_docker/env.properties
vi proxy_docker/app/config/config.properties
```

*Make sure user have same rpcuser and rpcpassword values as in bitcoin node's bitcoin.conf file (see below)*

```shell
vi proxy_docker/app/config/watcher_btcnode_curlcfg.properties
vi proxy_docker/app/config/spender_btcnode_curlcfg.properties
vi cron_docker/env.properties
vi pycoin_docker/env.properties
```

## Create cyphernode user, create proxy DB folder and build images

```shell
sudo useradd cyphernode
mkdir ~/btcproxydb ; sudo chown -R cyphernode:debian ~/btcproxydb ; sudo chmod g+ws ~/btcproxydb
mkdir -p ~/cyphernode-ssl/certs ~/cyphernode-ssl/private
openssl req -subj '/CN=localhost' -x509 -newkey rsa:4096 -nodes -keyout ~/cyphernode-ssl/private/key.pem -out ~/cyphernode-ssl/certs/cert.pem -days 365
docker build -t authapi api_auth_docker/.
docker build -t proxycronimg cron_docker/.
docker build -t btcproxyimg --build-arg USER_ID=$(id -u cyphernode) --build-arg GROUP_ID=$(id -g cyphernode) proxy_docker/.
docker build -t pycoinimg --build-arg USER_ID=$(id -u cyphernode) --build-arg GROUP_ID=$(id -g cyphernode) pycoin_docker/.
```

## Build images from Satoshi Portal's dockers repo
(For cyphernode, we are using host user cyphernode for all containers)

```shell
cd ..
git clone https://github.com/SatoshiPortal/dockers.git
cd dockers/x86_64/LN/c-lightning/
vi bitcoin.conf
```

*Make sure testnet, rpcuser and rpcpassword have the same value as in bitcoin node's bitcoin.conf file (see below)*

```console
rpcconnect=btcnode
rpcuser=rpc_username
rpcpassword=rpc_password
testnet=1
rpcwallet=ln01.dat
```

```shell
vi config
mkdir ~/.lightning
cp config ~/.lightning/
sudo chown -R cyphernode:debian ~/.lightning ; sudo chmod g+ws ~/.lightning
sudo find ~/.lightning -type d -exec chmod 2775 {} \; ; sudo find ~/.lightning -type f -exec chmod g+rw {} \;
docker build -t clnimg --build-arg USER_ID=$(id -u cyphernode) --build-arg GROUP_ID=$(id -g cyphernode) .
cd ../../bitcoin-core/
mkdir ~/.bitcoin
sudo chown -R cyphernode:debian ~/.bitcoin ; sudo chmod g+ws ~/.bitcoin
sudo find ~/.bitcoin -type d -exec chmod 2775 {} \; ; sudo find ~/.bitcoin -type f -exec chmod g+rw {} \;
docker build -t btcnode --build-arg USER_ID=$(id -u cyphernode) --build-arg GROUP_ID=$(id -g cyphernode) --build-arg CORE_VERSION="0.16.3" .
```

## Mount bitcoin data volume and make sure bitcoin configuration is ok
(I already had a bitcoin volume with blocks and chainstate folders sync'ed)
(Watcher and spender is the same bitcoin node, with different wallets)

```shell
sudo mount /dev/vdc ~/.bitcoin/
vi ~/.bitcoin/bitcoin.conf
```

*Make sure testnet, rpcuser and rpcpassword have the same value as in c-lightning node's bitcoin.conf file (see above)*

```console
testnet=1
txindex=1
rpcuser=rpc_username
rpcpassword=rpc_password
rpcallowip=10.0.0.0/24
#printtoconsole=1
maxmempool=64
dbcache=64
zmqpubrawblock=tcp://0.0.0.0:29000
zmqpubrawtx=tcp://0.0.0.0:29000
wallet=watching01.dat
wallet=spending01.dat
wallet=ln01.dat
walletnotify=curl cyphernode:8888/conf/%s
```

## Deploy the cyphernode stack

```shell
cd ~/cyphernode/
docker stack deploy --compose-file docker-compose.yml cyphernodestack
```

## Wait a few minutes and re-apply permissions

```shell
sudo chown -R cyphernode:debian ~/.lightning ; sudo chmod g+ws ~/.lightning
sudo chown -R cyphernode:debian ~/.bitcoin ; sudo chmod g+ws ~/.bitcoin
sudo find ~/.lightning -type d -exec chmod 2775 {} \; ; sudo find ~/.lightning -type f -exec chmod g+rw {} \;
sudo find ~/.bitcoin -type d -exec chmod 2775 {} \; ; sudo find ~/.bitcoin -type f -exec chmod g+rw {} \;
  ```

## Test the deployment

```shell
echo "GET /getbestblockinfo" | docker run --rm -i --network=cyphernodenet alpine nc cyphernode:8888 -
echo "GET /getbalance" | docker run --rm -i --network=cyphernodenet alpine nc cyphernode:8888 -
echo "GET /ln_getinfo" | docker run --rm -i --network=cyphernodenet alpine nc cyphernode:8888 -
docker exec -it `docker ps -q -f name=cyphernodestack_cyphernode` curl -H "Content-Type: application/json" -d "{\"pub32\":\"upub5GtUcgGed1aGH4HKQ3vMYrsmLXwmHhS1AeX33ZvDgZiyvkGhNTvGd2TA5Lr4v239Fzjj4ZY48t6wTtXUy2yRgapf37QHgt6KWEZ6bgsCLpb\",\"path\":\"0/25-30\"}" cyphernode:8888/derivepubpath
```
