### **1. Install required packages**

```
sudo apt update && \
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```

### **2. Install go**

```
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```

### **3.Build evmosd binary**

```
wget https://rpc-zero-gravity-testnet.win11strove.io/evmosd
chmod +x ./evmosd
mv ./evmosd /usr/local/bin/
evmosd version
```

### **4. Set up variables**

*Customize if you need*

```
echo 'export MONIKER="ChooseYourMonikerName"' >> ~/.bash_profile
echo 'export CHAIN_ID="zgtendermint_9000-1"' >> ~/.bash_profile
echo 'export WALLET_NAME="wallet"' >> ~/.bash_profile
echo 'export RPC_PORT="26657"' >> ~/.bash_profile
source $HOME/.bash_profile
```

> [!NOTE] 
> `MONIKER="ChooseYourMonikerName"` is a name you choice

### **5. Initialize the node**

```
cd $HOME
evmosd init $MONIKER --chain-id $CHAIN_ID
evmosd config chain-id $CHAIN_ID
evmosd config node tcp://localhost:$RPC_PORT
evmosd config keyring-backend os
```

### **6. Download genesis.json**

```
wget https://rpc-zero-gravity-testnet.win11strove.io/genesis.json -O $HOME/.evmosd/config/genesis.json
```

### **7. Add seeds and peers to the config.toml**

```
PEERS="1248487ea585730cdf5d3c32e0c2a43ad0cda973@peer-zero-gravity-testnet.win11strove.io:26326" && \
SEEDS="8c01665f88896bca44e8902a30e4278bed08033f@54.241.167.190:26656,b288e8b37f4b0dbd9a03e8ce926cd9c801aacf27@54.176.175.48:26656,8e20e8e88d504e67c7a3a58c2ea31d965aa2a890@54.193.250.204:26656,e50ac888b35175bfd4f999697bdeb5b7b52bfc06@54.215.187.94:26656" && \
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.evmosd/config/config.toml
```

### **8. Change ports (Optional)**

```
EXTERNAL_IP=$(wget -qO- eth0.me) \
PROXY_APP_PORT=26658 \
P2P_PORT=26656 \
PPROF_PORT=6060 \
API_PORT=1317 \
GRPC_PORT=9090 \
GRPC_WEB_PORT=9091
```

```bash
sed -i \
    -e "s/\(proxy_app = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$PROXY_APP_PORT\"/" \
    -e "s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$RPC_PORT\"/" \
    -e "s/\(pprof_laddr = \"\)\([^:]*\):\([0-9]*\).*/\1localhost:$PPROF_PORT\"/" \
    -e "/\[p2p\]/,/^\[/{s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$P2P_PORT\"/}" \
    -e "/\[p2p\]/,/^\[/{s/\(external_address = \"\)\([^:]*\):\([0-9]*\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/; t; s/\(external_address = \"\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/}" \
    $HOME/.evmosd/config/config.toml
```

```bash
sed -i \
    -e "/\[api\]/,/^\[/{s/\(address = \"tcp:\/\/\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$API_PORT\4/}" \
    -e "/\[grpc\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_PORT\4/}" \
    -e "/\[grpc-web\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_WEB_PORT\4/}" $HOME/.evmosd/config/app.toml
```

### **9. Configure prunning to save storage (Optional)**

```
sed -i.bak -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.evmosd/config/app.toml
```

### **10. Set min gas price**

```
sed -i "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.00252aevmos\"/" $HOME/.evmosd/config/app.toml

```

### **11. Enable indexer (Optional)**

```
sed -i "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.evmosd/config/config.toml

```

### **12. Create a service file**

```bash
sudo tee /etc/systemd/system/ogd.service > /dev/null << EOF
[Unit]
Description=OG Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which evmosd) start --home $HOME/.evmosd
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

### **13. Start the node**

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable ogd && \
sudo systemctl restart ogd && \
sudo journalctl -u ogd -f -o cat
```

### **14. Download snapshot**

```bash
wget https://rpc-zero-gravity-testnet.win11strove.io/latest_snapshot.tar.lz4
```

Stop the node

```bash
sudo systemctl stop ogd
```

Backup priv_validator_state.json

```bash
cp $HOME/.evmosd/data/priv_validator_state.json $HOME/.evmosd/priv_validator_state.json.backup
```

Reset DB

```bash
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd --keep-addr-book
```

Extract files fromt the arvhive

```bash
lz4 -d -c ./latest_snapshot.tar.lz4 | tar -xf - -C $HOME/.evmosd
```

Move priv_validator_state.json back

```bash
mv $HOME/.evmosd/priv_validator_state.json.backup $HOME/.evmosd/data/priv_validator_state.json
```

Restart the node

```bash
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```

then Ctrl C => next steps

### **15. Create a wallet for your validator**

```bash
evmosd keys add $WALLET_NAME
```

> [!NOTE] 
> Save seed pharse. If you don't save the seed pharse you can't get it back

if you want to recover wallet use command:

```bash
evmosd keys add $WALLET_NAME --recover
```

### **16. Extract the HEX address to request some tokens from the faucet**

```bash
echo "0x$(evmosd debug addr $(evmosd keys show $WALLET_NAME -a) | grep hex | awk '{print $3}')"
```

example: 0x94259.... This command will appear an address to use for faucet tokens

can extract the private keys in address above, use command:

```bash
evmosd keys unsafe-export-eth-key $WALLET_NAME
```

you can add account with private keys in metamask, account use faucet token if you want

> [!NOTE] 
> Save private keys add to metamask to faucet token

### **17. Faucet token**

[**https://faucet.0g.ai/**](https://faucet.0g.ai/)

> [!NOTE] 
> The faucet gives you *100000000000000000aevmos*. To make the validator join the active set you need at least *1000000000000000000aevmos* (**10 times more :(**)

### **18. Check wallet balances**

```bash
evmosd q bank balances $(evmosd keys show $WALLET_NAME -a)
```

if your node synced, pls check here

```
evmosd status | jq .SyncInfo.catching_up
```

if "catching_up"=False you can create validator

### **19. Create a validator**

```bash
evmosd tx staking create-validator \
--amount=10000000000000000aevmos \
--pubkey=$(evmosd tendermint show-validator) \
--moniker=$MONIKER \
--chain-id=$CHAIN_ID \
--commission-rate=0.05 \
--commission-max-rate=0.10 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=$WALLET_NAME \
--website="https://twitter...." \
--details="Stevendo handsome very" \
--gas=500000 --gas-prices=99999aevmos \
-y
```

please change variable your replace

- -website="your config"
- -details="your config"

You can check your Validator status here. Please allow some time for your validator to be added to the 0G Validator list: [**Here**](https://explorer.validatorvn.com/OG-Testnet/staking)

**backup validator keys**

```bash
cp $HOME/.evmosd/data/priv_validator_state.json $HOME/.evmosd/priv_validator_state.json.backup
```

**check after create validator**

```bash
evmosd q staking validator $(evmosd keys show $WALLET_NAME --bech val -a)
```
