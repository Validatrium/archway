# Info 

Created by [Validatrium](https://validatrium.com)

# Steps
- Setup node
- Run node
- Setup basic monitoring

# System requirements
- CPU: `4 Cores`
- RAM: `4 GB`
- SSD: `500 GB`
- System account: `root`


# Setup node

### Install prerequires: 

```bash
apt install jq

# download and install GO
wget https://go.dev/dl/go1.17.5.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.5.linux-amd64.tar.gz

echo '
export GOPATH=/usr/local/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN' >> $HOME/.bashrc
source $HOME/.bashrc

# compile binary from source code
git clone https://github.com/archway-network/archway
cd archway
git checkout v0.0.4 # newest v0.0.5 doesn't work right now

make install 
cd ~/

# create new sudo user
useradd archway  -mU -G sudo -s /usr/bin/bash
passwd archway 
su - archway
```

### Configure node
```bash
# replace <value> with your value
NODENAME=<your-name> # example NODENAME=Validator

# some usefull variables :D 
echo "
export GOPATH=/usr/local/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN

. <(archwayd completion)

export ACCOUNT=$NODENAME
export CHAIN=torii-1" >> $HOME/.bashrc
source $HOME/.bashrc

# init node
archwayd init $ACCOUNT --chain-id $CHAIN
# generate key. Save address and mnemonic from output!
archwayd keys add $ACCOUNT 

# download genesis
curl https://raw.githubusercontent.com/archway-network/testnets/main/torii-1/genesis.json -o ~/.archway/config/genesis.json

# set peers
PEERS=$(curl -s https://raw.githubusercontent.com/archway-network/testnets/main/torii-1/persistent_peers.txt)
sed -i.bak 's/persistent_peers = ../persistent_peers = "'$PEERS'"/' ~/.archway/config/config.toml

# set pruning options
sed -i.bak -e 's/pruning = "default"/pruning = "custom"/;s/pruning-keep-recent = "0"/pruning-keep-recent = "100"/;s/pruning-interval = "0"/pruning-interval = "10"/' $HOME/.archway/config/app.toml
```

### Run node
```bash
# manual start node
archwayd start


# run node as service:
sudo tee /etc/systemd/system/archwayd.service > /dev/null <<EOF 
[Unit]
Description=archway-node
After=network-online.target 

[Service]
User=archway
ExecStart=/usr/local/go/bin/archwayd start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target 
EOF

sudo systemctl enable archwayd
sudo systemctl start archwayd
journalctl -u archwayd -f
```

# Setup basic monitoring

### Monitoring for:
 - alert when disk usage is more than 80% and 90%
 - alert if height is not changing
 - alert if  node is not running 

### Prerequires
-   root account on server
-   telegram bot token
-   reciever telegram id

### Create telegram bot

In this guide we won't focus on creating telegram bot..
You can follow [this instruction](https://marketplace.creatio.com/sites/marketplace/files/app-guide/Instructions._Telegram_bot_1.pdf)

### Get reciever telegram id
Follow [this guide](https://www.wikihow.com/Know-Chat-ID-on-Telegram-on-Android#:~:text=Locate%20%22Chat.%22%20It's%20about,Last%20Name%2C%20and%20your%20Username.&text=Note%20the%20number%20next%20to,is%20your%20personal%20Chat%20ID) to get your telegram id
 
### Install
```bash
apt install -y monit jq

cd /root 
git clone https://github.com/Validatrium/archway.git
cd archway

echo include '/root/archway/conf/*' >> /etc/monit/monitrc

## IF YOU HAVE A DIFFERENT RPC PORT (edit 'height.sh' file, to recieve height notifications)
#replace with your rpc address: (example: localhost:26657)
RPC="localhost:26657"

# edit telegram.conf, replacce with your values to recieve alerts
TOKEN=<TOKEN>
CHATID=<YOUR-ID>

# check if you setup telegram notifications correctly: 
bin/sendtelegram -m 'hi there!' -c telegram.conf
# restart monitoring tool
service monit restart

```

# Additional section:

Get latest node height: 
`curl -s localhost:26657/status | jq .result.sync_info`

Get  node's peers count: 
`curl -s localhost:26657/net_info | jq '.result.peers | length'`

### Way to find more peers:

You can grap some peers from official RPC server
```bash
curl -s $RPC_ENDPOINT/net_info | jq  -r '.result.peers[].node_info | .id+"@"+.listen_addr'
```
and paste it into `$HOME/.archway/config/config.toml` 

as a `persistent_peers = "<peer-1>,<peer-2>"`
