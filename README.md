# Update if needed
sudo apt update && sudo apt upgrade -y

# Insall packages
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu -y

# Install GO
cd $HOME

ver="1.19.4"

wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"

sudo rm -rf /usr/local/go

sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"

rm "go$ver.linux-amd64.tar.gz"

echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile

source $HOME/.bash_profile

go version

#Celestia App build
cd $HOME

git clone https://github.com/celestiaorg/celestia-app.git

cd celestia-app

git checkout v0.11.0

make install

# check
celestia-appd version

# OUTPUT
# 0.11.0

cd $HOME

git clone https://github.com/celestiaorg/networks

#Init node
# set your names
CELESTIA_NODENAME="MY_NODE"

CELESTIA_WALLET="MY_WALLET"

CELESTIA_CHAIN="mocha"  # do not change

# save vars
echo "export CELESTIA_CHAIN=$CELESTIA_CHAIN

export CELESTIA_NODENAME=${CELESTIA_NODENAME}

export CELESTIA_WALLET=${CELESTIA_WALLET}

" >> $HOME/.bash_profile

source $HOME/.bash_profile

# do init
celestia-appd init $CELESTIA_NODENAME --chain-id $CELESTIA_CHAIN

# OUTPUT EXAMPLE
{"app_message":{"auth":{"accounts":[],"params":{"max_memo_characters":"256","sig_verify_cost_ed25519":"590","sig_verify_cost_secp256k1":"1000","tx_sig_limit":"7","tx_size_cost_per_byte":"10"}},"bank":{"balances":[],"denom_metadata":[{"base":"utia","denom_units":[{"aliases":["microtia"],"denom":"utia","exponent":0},{"aliases":[],"denom":"TIA","exponent":6}],"description":"The native staking token of the Celestia network.","display":"TIA","name":"TIA","symbol":"TIA","uri":"","uri_hash":""}],"params":{"default_send_enabled":true,"send_enabled":[]},"supply":[]},"capability":{"index":"1","owners":[]},"crisis":{"constant_fee":{"amount":"1000","denom":"utia"}},"distribution":{"delegator_starting_infos":[],"delegator_withdraw_infos":[],"fee_pool":{"community_pool":[]},"outstanding_rewards":[],"params":{"base_proposer_reward":"0.010000000000000000","bonus_proposer_reward":"0.040000000000000000","community_tax":"0.020000000000000000","withdraw_addr_enabled":true},"previous_proposer":"","validator_accumulated_commissions":[],"validator_current_rewards":[],"validator_historical_rewards":[],"validator_slash_events":[]},"evidence":{"evidence":[]},"feegrant":{"allowances":[]},"genutil":{"gen_txs":[]},"gov":{"deposit_params":{"max_deposit_period":"172800s","min_deposit":[{"amount":"10000000","denom":"stake"}]},"deposits":[],"proposals":[],"starting_proposal_id":"1","tally_params":{"quorum":"0.334000000000000000","threshold":"0.500000000000000000","veto_threshold":"0.334000000000000000"},"votes":[],"voting_params":{"voting_period":"172800s"}},"mint":{"minter":{"annual_provisions":"0.000000000000000000","inflation":"0.130000000000000000"},"params":{"blocks_per_year":"6311520","goal_bonded":"0.670000000000000000","inflation_max":"0.200000000000000000","inflation_min":"0.070000000000000000","inflation_rate_change":"0.130000000000000000","mint_denom":"utia"}},"params":null,"payment":{},"qgb":{"params":{"data_commitment_window":"400"}},"slashing":{"missed_blocks":[],"params":{"downtime_jail_duration":"600s","min_signed_per_window":"0.500000000000000000","signed_blocks_window":"100","slash_fraction_double_sign":"0.050000000000000000","slash_fraction_downtime":"0.010000000000000000"},"signing_infos":[]},"staking":{"delegations":[],"exported":false,"last_total_power":"0","last_validator_powers":[],"params":{"bond_denom":"utia","historical_entries":10000,"max_entries":7,"max_validators":100,"min_commission_rate":"0.000000000000000000","unbonding_time":"1814400s"},"redelegations":[],"unbonding_delegations":[],"validators":[]},"upgrade":{},"vesting":{}},"chain_id":"mocha","gentxs_dir":"","moniker":"MZONDER","node_id":"29d919a4fd8381a20f5cf815f93e1f64c2ca9a85"}


# copy genesis
cp $HOME/networks/mocha/genesis.json $HOME/.celestia-app/config/

# reset
celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app

#Config peers and seeds

SEEDS="9aa8a73ea9364aa3cf7806d4dd25b6aed88d8152@celestia.seed.mzonder.com:11156"
sed -i "s|^seeds *=.*|seeds = \"$SEEDS\"|" $HOME/.celestia-app/config/config.toml

# it takes about 5 min to find peers, be patient

#Config pruning and snapshots 
# pruning and snapshots

pruning_keep_recent="10000"

pruning_interval=$(shuf -n1 -e 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97)

snapshot_interval="5000"

sed -i "s/^pruning *=.*/pruning = \"custom\"/;\

s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/;\

s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/;\

s/^snapshot-interval *=.*/snapshot-interval = $snapshot_interval/" $HOME/.celestia-app/config/app.toml

#Config client
celestia-appd config chain-id $CELESTIA_CHAIN --home $HOME/.celestia-app

celestia-appd config keyring-backend test

#Create and run service
# create service
tee $HOME/celestia-appd.service > /dev/null <<EOF

[Unit]

Description=celestia-appd

After=network-online.target

[Service]

User=$USER

ExecStart=$(which celestia-appd) start

Restart=on-failure

RestartSec=3

LimitNOFILE=65535

[Install]

WantedBy=multi-user.target

EOF

sudo mv $HOME/celestia-appd.service /etc/systemd/system/

sudo systemctl enable celestia-appd

sudo systemctl daemon-reload

sudo systemctl restart celestia-appd && journalctl -u celestia-appd -f -o cat

# Press CTRL+C to interrupt logs output

# Before continue make sure you are fully synced with the network

curl -s localhost:26657/status

# should be "catching_up": false

# recover key if participated in MAMAKI

celestia-appd keys add $CELESTIA_WALLET --recover

# OR create new one: celestia-appd keys add $CELESTIA_WALLET

# save addr and valoper to vars (optional)

CELESTIA_ADDR=$(celestia-appd keys show $CELESTIA_WALLET -a)

echo $CELESTIA_ADDR

echo 'export CELESTIA_ADDR='${CELESTIA_ADDR} >> $HOME/.bash_profile

CELESTIA_VALOPER=$(celestia-appd keys show $CELESTIA_WALLET --bech val -a)

echo $CELESTIA_VALOPER

echo 'export CELESTIA_VALOPER='${CELESTIA_VALOPER} >> $HOME/.bash_profile

source $HOME/.bash_profile


# Check balance of validator wallet

celestia-appd q bank balances $CELESTIA_ADDR

# you should see your balance from mamaki testnet

# Create orchestrator key

celestia-appd keys add ORCHESTRATOR_ADDRESS

**# !!!! SAVE MNEMONIC FROM OUTPUT !!!!**

# save addr to var
****ORCHESTRATOR_ADDRESS=$(celestia-appd keys show ORCHESTRATOR_ADDRESS -a)
****echo $ORCHESTRATOR_ADDRESS
echo 'export ORCHESTRATOR_ADDRESS='${ORCHESTRATOR_ADDRESS} >> $HOME/.bash_profile

## Create new ETH address in metamask

# set var
EVM_ADDRESS=0x12345xxxxxxxxxxxYOUR_ETH_ADDRxxxx
