Create validator
# check vars

echo $CELESTIA_NODENAME,$CELESTIA_CHAIN,$CELESTIA_WALLET,$ORCHESTRATOR_ADDRESS,$EVM_ADDRESS | tr "," "\n" | nl 

# create validator
celestia-appd tx staking create-validator \
--amount=1000000utia \
--pubkey=$(celestia-appd tendermint show-validator) \
--moniker=$CELESTIA_NODENAME \
--chain-id=$CELESTIA_CHAIN \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1000000 \
--from=$CELESTIA_WALLET \
--evm-address=$EVM_ADDRESS \
--orchestrator-address=$ORCHESTRATOR_ADDRESS \
--fees 5000utia \
--gas auto \
--gas-adjustment 1.5


# check validator (1 min after creating)

celestia-appd q staking validator $CELESTIA_VALOPER


#Download and save "$HOME/.celestia-app/config/priv_validator_key.json" to your PC!


# delegate more if needed

celestia-appd tx staking delegate $CELESTIA_VALOPER 10000000utia --from=$CELESTIA_WALLET --fees 300utia



# output active set

celestia-appd q staking validators --limit=3000 -oj \
 | jq -r '.validators[] | select(.status=="BOND_STATUS_BONDED") | [(.tokens|tonumber / pow(10;6)), .description.moniker] | @csv' \
 | column -t -s"," | tr -d '"'| sort -k1 -n -r | nl

# edit avatar if needed

celestia-appd tx staking edit-validator --identity "YOUR_KEYBASE_ID" --from $CELESTIA_WALLET --fees 300utia
