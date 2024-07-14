# AlloraNetwork-Worker-node


![allo](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/3ed13d4e-b80a-43c6-bdcb-d068e033af3e)


## Lets Start

## System Requirements
- OS: Windows, macOS, or Linux
- CPU: Minimum of 1/2 core
- Memory: 2 to 4 GB
- Storage: SSD or NVMe with at least 5GB of space

#### Install packages & dependencies 

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install ca-certificates zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev curl git wget make jq build-essential pkg-config lsb-release libssl-dev libreadline-dev libffi-dev gcc screen unzip lz4 -y
```

--------------
#### Install python3 
- Press ```y``` and proceed
  
```
sudo apt install python3
python3 --version

sudo apt install python3-pip
pip3 --version
```

![python](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/15fad6ce-8e62-4623-b8d6-a0d972df4d17)



#### Install Docker 

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
docker version
```


#### Install Docker compose 

```
VER=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)

curl -L "https://github.com/docker/compose/releases/download/"$VER"/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

![docker](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/f4a845d9-8eb2-4e37-89e2-df9e80085ddb)



#### Docker permission 

```
sudo groupadd docker
sudo usermod -aG docker $USER
```


#### Install GO 

```
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

![go](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/515f2197-5f5f-46ad-9bd8-9ee2a6f32120)



#### Install Allorad: Wallet 
```
git clone https://github.com/allora-network/allora-chain.git
cd allora-chain && make all
allorad version
```


#### Create Wallet 
- Create a new wallet & save mnemonic
  
```
allorad keys add testkey
```
or 

- import ur existing Keplr wallet or Leap wallet with seed phrase

```
allorad keys add testkey --recover
```


---------------------
- Add AlloraNetwork on Keplr wallet 
Link : https://explorer.edgenet.allora.network/wallet/suggest

- Copy AlloraNetwork Address
Link : https://app.allora.network/points/campaigns

- faucet
Link : https://faucet.edgenet.allora.network/

![add kep](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/3626c641-4845-40ae-a9a8-b6bf21a4a927)

![faucet](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/a7fb5678-4025-438e-928e-4fc01cb089d6)



#### Install Worker 

```
git clone https://github.com/allora-network/basic-coin-prediction-node

cd basic-coin-prediction-node
```
```
mkdir worker-data && mkdir head-data
```


#### permissions

```
sudo chmod -R 777 worker-data
sudo chmod -R 777 head-data
```


#### Create Head key 
```
sudo docker run -it --entrypoint=bash -v ./head-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
```



#### Create Worker key 
```
sudo docker run -it --entrypoint=bash -v ./worker-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
```

#### Save head ID 
```
cat head-data/keys/identity
```

![head id](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/986a48db-7a77-4599-b837-40876a4ff313)



## Lets Connect to Allora Chain 
```
rm -rf docker-compose.yml && nano docker-compose.yml
```

#### Copy below complete code and paste into terminal 
Replace with ur ```head-id``` and ```wallet_seed_phrase```

```
version: '3'

services:
  inference:
    container_name: inference-basic-eth-pred
    build:
      context: .
    command: python -u /app/app.py
    ports:
      - "8000:8000"
    networks:
      eth-model-local:
        aliases:
          - inference
        ipv4_address: 172.22.0.4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/inference/ETH"]
      interval: 10s
      timeout: 5s
      retries: 12
    volumes:
      - ./inference-data:/app/data

  updater:
    container_name: updater-basic-eth-pred
    build: .
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
    command: >
      sh -c "
      while true; do
        python -u /app/update_app.py;
        sleep 24h;
      done
      "
    depends_on:
      inference:
        condition: service_healthy
    networks:
      eth-model-local:
        aliases:
          - updater
        ipv4_address: 172.22.0.5

  worker:
    container_name: worker-basic-eth-pred
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9011 \
          --boot-nodes=/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=allora-topic-1-worker \
          --allora-chain-key-name=testkey \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.edgenet.allora.network/ \
          --allora-chain-topic-id=1
    volumes:
      - ./worker-data:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker
        ipv4_address: 172.22.0.10

  head:
    container_name: head-basic-eth-pred
    image: alloranetwork/allora-inference-base-head:latest
    environment:
      - HOME=/data
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        allora-node --role=head --peer-db=/data/peerdb --function-db=/data/function-db  \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9010 --rest-api=:6000
    ports:
      - "6000:6000"
    volumes:
      - ./head-data:/data
    working_dir: /data
    networks:
      eth-model-local:
        aliases:
          - head
        ipv4_address: 172.22.0.100


networks:
  eth-model-local:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

volumes:
  inference-data:
  worker-data:
  head-data:
```

![code](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/0a134caa-d696-42ab-bf83-5f0293070ae3)



- Save YML File ```CTRL X + Y,``` then ```ENTER```


#### Employ Worker 
```
docker compose build
docker compose up -d
```

- it takes some time : Sit and Relax Take a cup of coffee 



#### Check ur Node Status & Copy CONTAINER ID

```
docker ps
```

![docker ps](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/201e5c28-b2be-46d8-af42-7c506e85ea6b)



#### Replace with ur  ```CONTAINER-ID```

```
docker logs -f CONTAINER-ID
```


- Success: register node Tx Hash : 1df4gbdr5g1fw5gs1b6dt5hd1hd6it5sg1sdgs5s6g4s5q1gssdgsr7f5gg0000000

![hash](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/6bb0ddbe-60e2-40a7-bc81-a3377d58a4ba)



#### Check ur Node Status just copy Entire code & paste :if  Response #200 ur good to go

```
curl --location 'http://localhost:6000/api/v1/functions/execute' \
--header 'Content-Type: application/json' \
--data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "1",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 2
    }
}'
```



### update node 

```
curl http://localhost:8000/update
```

## Check Inference node 
```
curl http://localhost:8000/inference/ETH
```

![allora](https://github.com/CryptoReddys/AlloraNetwork-node-setup/assets/59482451/37f7334e-baa7-4bb6-8155-fc1e37e117ee)




- Thats it Now Ur Node Running Good & Healthy, if any changes made i will update HERE : Always #DYOR 
