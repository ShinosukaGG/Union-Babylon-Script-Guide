# 🎯 Union Testnet (Babylon -> Xion) Auto Transaction Bot

A complete setup guide for Babylon CLI and a bash script that automates transaction sending on testnet. Works on any Ubuntu-based VPS or WSL.

---

## 🔥 One-Shot Setup Guide

### 🤖 System Preparation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl iptables build-essential git wget jq make gcc nano automake autoconf tmux htop pkg-config libssl-dev tar clang unzip openssl -y
```

---

### 🤖 Install Go

```bash
cd ~
ver="1.22.3"
wget https://go.dev/dl/go$ver.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go$ver.linux-amd64.tar.gz

echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
go version
```

---

### 🤖 Install Babylon CLI

```bash
cd ~
git clone https://github.com/babylonlabs-io/babylon.git
cd babylon
git checkout v2.0.0-rc.0
make install

echo 'export PATH=$HOME/go/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

which babylond
babylond version
```

---

### 🔗 Create Wallet

```bash
babylond keys add wallet --recover --keyring-backend test
```

*Paste your mnemonic when prompted, or remove `--recover` to create a new one.*
*Press `ENTER`*
---

### ✅ Create Transaction Script

```bash
nano babylon.sh
```

Paste the following:

```bash
#!/bin/bash

CHAIN_ID="bbn-test-5"
NODE="https://babylon-testnet-rpc.polkachu.com:443"
SENDER="wallet"
CONTRACT="bbn1336jj8ertl8h7rdvnz4dh5rqahd09cy0x43guhsxx6xyrztx292q77945h"
AMOUNT="100ubbn"
FEES="900ubbn"

while true
do
    BAL=$(babylond q bank balances $(babylond keys show $SENDER -a --keyring-backend test) --node $NODE --output json | jq -r '.balances[] | select(.denom=="ubbn") | .amount')

    if [[ -z "$BAL" || "$BAL" -lt "$AMOUNT" ]]; then
        echo "❌ Not enough balance. Current: ${BAL:-0} ubbn"
        sleep 10
        continue
    fi

    SALT="0x$(openssl rand -hex 32)"
    NOW_NS=$(date +%s%N)
    TIMEOUT_TS=$((NOW_NS + 600000000000))

    if [ ! -f instruction.hex ]; then
        echo "❌ instruction.hex file missing!"
        exit 1
    fi

    RAW_HEX=$(tr -d '\n\r ' < instruction.hex)
    RAW_HEX="${RAW_HEX#0x}"
    INSTRUCTION_HEX="0x$RAW_HEX"

    MSG=$(jq -nc \
      --arg channel_id 4 \
      --arg timeout_height 0 \
      --arg timeout_timestamp "$TIMEOUT_TS" \
      --arg salt "$SALT" \
      --arg instruction "$INSTRUCTION_HEX" \
    '{
      send: {
        channel_id: ($channel_id | tonumber),
        timeout_height: ($timeout_height | tostring),
        timeout_timestamp: ($timeout_timestamp | tostring),
        salt: $salt,
        instruction: $instruction
      }
    }')

    echo "?? Sending tx (SALT: $SALT)"

    babylond tx wasm execute "$CONTRACT" "$MSG" \
      --amount "$AMOUNT" \
      --fees "$FEES" \
      --sign-mode direct \
      --from "$SENDER" \
      --chain-id "$CHAIN_ID" \
      --node "$NODE" \
      --keyring-backend test \
      --gas auto --gas-adjustment 1.2 -y

    echo "⏳ Waiting 10 seconds..."
    sleep 10
done
```

*Press `CTRL+O` then `ENTER`*
*Press `CTRL+X`*

Make it executable:

```bash
chmod +x babylon.sh
```

---

### 🔗 Create `instruction.hex`

1. Go to: [https://app.union.build/transfer](https://app.union.build/transfer)  
2. Complete a transfer → check explorer logs: [https://testnet.babylon.explorers.guru/](https://testnet.babylon.explorers.guru/)
   ![Preview](./IMG_20250619_233501.jpg)
3. Find the `instruction` hex and save it:

```bash
echo "0xYOUR_INSTRUCTION_HEX" > instruction.hex
```

---

### ▶️ Run the Bot

```bash
bash babylon.sh
```

---

✅ Working Script for Union Testnet Transactions (Babylon -> Xion), Completes upto 1000 transactions within 3hours with just 1tBaby (faucet).

📝 If any error/feedback, dm me @Shinosuka_eth on Telegram, X (Twitter), or Discord. 
