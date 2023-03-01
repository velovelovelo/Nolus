# Nolus
<p align="center">
 <img height="200" height="auto" src="https://user-images.githubusercontent.com/34649601/207593974-32d7cb69-eca9-4096-bc96-246fe7038c88.png">
---

 
 Minimum requirements
- 2+ vCPU
- 4+ GB RAM
- 120+ GB SSD
---

# Official Links
### [Official Document](https://docs-nolus-protocol.notion.site/Run-a-Full-Node-7a92545223e7483bb4a02cce30b53aa8)
### [Nolus Official Discord](https://discord.gg/nolus-protocol)
 
 **Chain ID**: nolus-rila

### Public Endpoint

>- API : https://api.nolus.velochan.xyz
>- RPC : https://rpc.nolus.velochan.xyz
>- gRPC : https://grpc.nolus.velochan.xyz

# Explorer
### https://explorer-rila.nolus.io/nolus-rila

 # Manual Installation
### Setup validator name


Replace **YOUR_MONIKER_GOES_HERE** with your validator name


```bash
MONIKER="YOUR_MONIKER_GOES_HERE"
```

### Install dependencies

#### Update system and install build tools

```bash
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```

#### Install Go

```bash
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.19.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

### Download and build binaries

```bash
# Clone project repository
cd $HOME
rm -rf nolus-core
git clone https://github.com/Nolus-Protocol/nolus-core.git
cd nolus-core
git checkout v0.1.43

# Build binaries
make build

# Prepare binaries for Cosmovisor
mkdir -p $HOME/.nolus/cosmovisor/genesis/bin
mv target/release/nolusd $HOME/.nolus/cosmovisor/genesis/bin/
rm -rf build

# Create application symlinks
ln -s $HOME/.nolus/cosmovisor/genesis $HOME/.nolus/cosmovisor/current
sudo ln -s $HOME/.nolus/cosmovisor/current/bin/nolusd /usr/local/bin/nolusd
```

### Install Cosmovisor and create a service

```bash
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0

# Create service
sudo tee /etc/systemd/system/nolusd.service > /dev/null << EOF
[Unit]
Description=nolus-testnet node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.nolus"
Environment="DAEMON_NAME=nolusd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.nolus/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nolusd
```

### Initialize the node

```bash
# Set node configuration
nolusd config chain-id nolus-rila
nolusd config keyring-backend test
nolusd config node tcp://localhost:43657

# Initialize the node
nolusd init $MONIKER --chain-id nolus-rila

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/nolus-testnet/genesis.json > $HOME/.nolus/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/nolus-testnet/addrbook.json > $HOME/.nolus/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@nolus-testnet.rpc.kjnodes.com:43659\"|" $HOME/.nolus/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.0025unls\"|" $HOME/.nolus/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.nolus/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:43658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:43657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:43060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:43656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":43660\"%" $HOME/.nolus/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:43317\"%; s%^address = \":8080\"%address = \":43080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:43090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:43091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:43545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:43546\"%" $HOME/.nolus/config/app.toml
```

### Download latest chain snapshot from kjnodes

```bash
curl -L https://snapshots.kjnodes.com/nolus-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.nolus
[[ -f $HOME/.nolus/data/upgrade-info.json ]] && cp $HOME/.nolus/data/upgrade-info.json $HOME/.nolus/cosmovisor/genesis/upgrade-info.json
```

### Start service and check the logs

```bash
sudo systemctl start nolusd && sudo journalctl -u nolusd -f --no-hostname -o cat
```

##  Wallet

#### Add new key

```bash
nolusd keys add wallet
```

#### Recover existing key

```bash
nolusd keys add wallet --recover
```

#### List all keys

```bash
nolusd keys list
```

#### Delete key

```bash
nolusd keys delete wallet
```

#### Export key to the file

```bash
nolusd keys export wallet
```

#### Import key from the file

```bash
nolusd keys import wallet wallet.backup
```

#### Query wallet balance

```bash
nolusd q bank balances $(nolusd keys show wallet -a)
```

### Before create validator make sure your node already sync
#### Create validator

```bash
nolusd tx staking create-validator \
--amount 1000000unls \
--pubkey $(nolusd tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id nolus-rila \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--fees 700unls \
-y
```

#### Edit existing validator

```bash
nolusd tx staking edit-validator \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL"
--chain-id nolus-rila \
--commission-rate 0.05 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--fees 700unls \
-y
```
# Upgrade
**Chain ID**: nolus-rila | **Latest Version Tag**: v0.1.43 | **Custom Port**: 43
```bash cd $HOME
rm -rf nolus-core
git clone https://github.com/Nolus-Protocol/nolus-core.git
cd nolus-core
git checkout v0.1.43
make build
```

#prepare binaries

```bash 
mkdir -p $HOME/.nolus/cosmovisor/upgrades/v0.1.43/bin
mv target/release/nolusd $HOME/.nolus/cosmovisor/upgrades/v0.1.43/bin/
rm -rf build
```

when upgrade block height is reached, Cosmovisor will handle it automatically!*

#### Unjail validator

```bash
nolusd tx slashing unjail --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Jail reason

```bash
nolusd query slashing signing-info $(nolusd tendermint show-validator)
```

#### List all active validators

```bash
nolusd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```


#### View validator details

```bash
nolusd q staking validator $(nolusd keys show wallet --bech val -a)
```

## Delegate-Withdraw

#### Withdraw rewards from all validators

```bash
nolusd tx distribution withdraw-all-rewards --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Withdraw commission and rewards from your validator

```bash
nolusd tx distribution withdraw-rewards $(nolusd keys show wallet --bech val -a) --commission --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Delegate tokens to yourself

```bash
nolusd tx staking delegate $(nolusd keys show wallet --bech val -a) 1000000unls --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Delegate tokens to validator

```bash
nolusd tx staking delegate <TO_VALOPER_ADDRESS> 1000000unls --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Redelegate tokens to another validator

```bash
nolusd tx staking redelegate $(nolusd keys show wallet --bech val -a) <TO_VALOPER_ADDRESS> 1000000unls --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Unbond tokens from your validator

```bash
nolusd tx staking unbond $(nolusd keys show wallet --bech val -a) 1000000unls --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Send tokens to the wallet

```bash
nolusd tx bank send wallet <TO_WALLET_ADDRESS> 1000000unls --from wallet --chain-id nolus-rila
```

## Vote Governance

#### List all proposals

```bash
nolusd query gov proposals
```

#### View proposal by id

```bash
nolusd query gov proposal 1
```

#### Vote 'Yes'

```bash
nolusd tx gov vote 1 yes --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Vote 'No'

```bash
nolusd tx gov vote 1 no --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Vote 'Abstain'

```bash
nolusd tx gov vote 1 abstain --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```

#### Vote 'NoWithVeto'

```bash
nolusd tx gov vote 1 nowithveto --from wallet --chain-id nolus-rila --gas-adjustment 1.4 --gas auto --fees 500unls -y
```



## Check validator status

#### Get validator info

```bash
nolusd status 2>&1 | jq .ValidatorInfo
```

#### Get sync info

```bash
curl -s localhost:${PORT}657/status | jq .result.sync_info
```

#### Get node peer

```bash
echo $(nolusd tendermint show-node-id)'@'$(curl -s ifconfig.me)':'$(cat $HOME/.nolus/config/config.toml | sed -n '/Address to listen for incoming connection/{n;p;}' | sed 's/.*://; s/".*//')
```

#### Reset chain data

```bash
nolusd tendermint unsafe-reset-all --home $HOME/.nolus --keep-addr-book
```

#### Remove node

Please, Make sure you have backed up your **priv_validator_key.json**!


```bash
cd $HOME
sudo systemctl stop nolusd
sudo systemctl disable nolusd
sudo rm /etc/systemd/system/nolusd.service
sudo systemctl daemon-reload
rm -f $(which nolusd)
rm -rf $HOME/.nolus
rm -rf $HOME/nolus-core
```

## Usefull command

#### Reload service configuration

```bash
sudo systemctl daemon-reload
```

#### Enable service

```bash
sudo systemctl enable nolusd
```

#### Disable service

```bash
sudo systemctl disable nolusd
```

#### Start service

```bash
sudo systemctl start nolusd
```

#### Stop service

```bash
sudo systemctl stop nolusd
```

#### Restart service

```bash
sudo systemctl restart nolusd
```

#### Check service status

```bash
sudo systemctl status nolusd
```

#### Check service logs

```bash
sudo journalctl -u nolusd -f --no-hostname -o cat
```
#### Snapshot

```bash
sudo systemctl stop nolusd
 ```
cp $HOME/.nolus/data/priv_validator_state.json $HOME/.nolus/priv_validator_state.json.backup
rm -rf $HOME/.nolus/data
 ```
curl -L https://snapshot.nolus.velochan.xyz/snapshots/nolus/nolus-snapshot-20230301.tar.lz4   | lz4 -dc - | tar -xf - -C $HOME/.nolus
mv $HOME/.nolus/priv_validator_state.json.backup $HOME/.nolus/data/priv_validator_state.json
```
sudo systemctl restart nolusd && journalctl -u nolusd -f --no-hostname -o cat
```
