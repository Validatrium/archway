# Info 

Created by [Validatrium](https://validatrium.com)

This guide is about how to run archway node as a common cosmos-sdk based app. 
That's the reason we're running node as a service. 

# Links
- explorer: https://explorer.augusta-1.archway.tech/
- rpc: https://rpc.augusta-1.archway.tech/
- snapshot: https://snap.validatrium.club/archway/archway-620000-pruned.tar

# Steps
- Setup node
- Download snapshot
- Run node
- Create Validator
- Setup basic monitoring
- Additional section

# System requirements
- CPU: `4 Cores`
- RAM: `4 GB`
- SSD: `500 GB`
- System account: `root`


# Setup node
```bash
# prerequires
sudo apt install  jq
# install docker 
curl -sL get.docker.com | sudo bash

# set name for your node
# replace <value> with your value.
NODENAME=<your-node-name> #example NODENAME=Validatrium

echo "
alias archwayd='docker run --rm -it --network host -v ~/.archway:/root/.archway archwaynetwork/archwayd:augusta' 

RPC_ENDPOINT=https://rpc.augusta-1.archway.tech

export ACCOUNT=$NODENAME
export CHAIN=augusta-1
" >> ~/.bashrc
source ~/.bashrc

# init node and create key
archwayd init $ACCOUNT --chain-id $CHAIN
# do not forget to save a mnemonic! It's the only way to recover your wallet
archwayd keys add $ACCOUNT

# or recover key if you already have one
archwayd keys add $ACCOUNT --recover

# download genesis
curl -s "$RPC_ENDPOINT/genesis" | jq '.result.genesis' > ~/.archway/config/genesis.json

# insert seed and peers
SEED=2f234549828b18cf5e991cc884707eb65e503bb2@34.74.129.75:31076
PEERS=0d7facb555de00a61f28420e6654a4039c99f63b@23.88.37.54:26656,bd985a14ebbcfa5b73794a1e77d888ec96e940a1@5.9.199.71:26656

sed -i.bak -e's/seeds =*.*/seeds = "'$SEED'"/;s/persistent_peers =*.*/persistent_peers = "'$PEERS'"/' $HOME/.archway/config/config.toml

# set pruning options
sed -i.bak -e 's/pruning = "default"/pruning = "custom"/;s/pruning-keep-recent = "0"/pruning-keep-recent = "100"/;s/pruning-interval = "0"/pruning-interval = "10"/' $HOME/.archway/config/app.toml
```

# Download snapshot
If you don't want to wait till node is fully synced you can download our snapshot. It's on height ~620000. 

```bash
# be sure node is down!
rm -rf $HOME/.archway/data
# snapshot size is 24GB, it might take a while till download completed
wget https://snap.validatrium.club/archway/archway-620000-pruned.tar
tar -xvf archway-620000-pruned.tar -C .archway/
```

# Run node
Manual run: 
```bash
archwayd start --x-crisis-skip-assert-invariants
```

Run as service 
```bash
sudo tee /etc/systemd/system/archwayd.service > /dev/null <<EOF 
[Unit]
Description=archwayd node
After=network-online.target 

[Service]
User=root
ExecStart=/bin/bash -c "/usr/bin/docker run --rm --network host -v ~/.archway:/root/.archway archwaynetwork/archwayd:augusta start --x-crisis-skip-assert-invariants"
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target 
EOF


systemctl enable archwayd
systemctl start archwayd
# logs
journalctl -u archwayd -f
```

# Create validator

After your node is full synced run: 

```bash
archwayd tx staking create-validator --amount 0uaugust --from $ACCOUNT --commission-max-change-rate "0.01" --commission-max-rate "0.1" --commission-rate "0.01" --min-self-delegation "1" --pubkey $(archwayd tendermint show-validator) --moniker $ACCOUNT ---chain-id $CHAIN --gas 300000 --fees 3uaugust
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
