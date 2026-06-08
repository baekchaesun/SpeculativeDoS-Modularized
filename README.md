# SpecDOS Physical Testnet

공격자(tx 생성/전송자)와 edge node(block 생성/검증자)를 물리적으로 분리하기 위한 실행 가이드.

```text
Tx Generator
  - honest tx 생성
  - attack tx 생성
  - edge node RPC로 전송

Edge nodes
  - validation + block generation
```

중요: edge node는 일반 geth가 아니라 이 repo의 modified geth/builder fork로 실행할 것.

## Build

```bash
cd builder
go build ./cmd/specdos
```

## Modified Files

수정된 파일만 commit/push하려면 아래 파일만 stage하면 됩니다.

```text
SpeculativeDoS-main/
├── .gitignore
└── builder/
    ├── cmd/
    │   └── specdos/
    │       └── main.go
    ├── docs/
    │   └── specdos-physical-testnet.md
    └── specdos/
        ├── config.go
        ├── monitor.go
        └── txgen.go
```

## 1. Lab Config 생성

```bash
./specdos init \
  --out ./lab \
  --validators 3 \
  --honest 5120 \
  --attackers 5120 \
  --chain-id 13371337 \
  --clique-period 12
```

생성 파일:

```text
./lab/genesis.json
./lab/keys.json
```

`keys.json`에는 private key가 평문 작성됨. public chain에 사용하지 말고 git에 올리지 말 것.

## 2. Edge Node 실행

validator 0:

```bash
./specdos node \
  --genesis ./lab/genesis.json \
  --keys ./lab/keys.json \
  --validator-index 0 \
  --datadir ./node0 \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --p2p.addr 0.0.0.0 \
  --p2p.port 30303
```

validator 1 이상은 앞 node의 `enode`를 `--staticnodes`에 삽입.

```bash
./specdos node \
  --genesis ./lab/genesis.json \
  --keys ./lab/keys.json \
  --validator-index 1 \
  --datadir ./node1 \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --p2p.addr 0.0.0.0 \
  --p2p.port 30303 \
  --staticnodes "enode://..."
```

원본 테스트의 censorship blocklist를 켜려면:

```bash
--verify-censorship
```

## 3. Tx Generator 실행

Honest only:

```bash
./specdos txgen \
  --keys ./lab/keys.json \
  --rpc http://EDGE_NODE_IP:8545 \
  --mode honest
```

ConditionalExhaust:

```bash
./specdos txgen \
  --keys ./lab/keys.json \
  --rpc http://EDGE_NODE_IP:8545 \
  --mode conditional \
  --attacker-accounts 140 \
  --attacker-rate 2 \
  --attacker-chunk 140
```

Combined MemPurge + ConditionalExhaust:

```bash
./specdos txgen \
  --keys ./lab/keys.json \
  --rpc http://EDGE_NODE_IP:8545 \
  --mode combined \
  --mem-purge-len 2 \
  --attacker-accounts 140 \
  --attacker-rate 2 \
  --attacker-chunk 140 \
  --contract-out ./lab/contract-address.txt
```

의미:

```text
attacker-accounts = 공격자 계정 수
attacker-rate     = 초당 전송 chunk 수
attacker-chunk    = chunk 하나에 담는 공격 tx 수
```

총 공격 tx 수:

```text
conditional = attacker-accounts * attacker-txs-per-account
combined    = attacker-accounts * mem-purge-len
```

## 4. Monitor 실행

block별 tx 포함 결과를 JSONL로 기록.

```bash
./specdos monitor \
  --keys ./lab/keys.json \
  --rpc http://EDGE_NODE_IP:8545 \
  --start latest \
  --interval 1s \
  --out ./specdos-metrics-edge0.jsonl
```

기록 항목:

```text
block number
tx count
honest tx count
attacker tx count
empty block 여부
gas used
```

## Network

권장 포트:

```text
8545/TCP       AWS txgen -> edge node RPC
30303/TCP+UDP  edge node <-> edge node P2P
22/TCP         관리용 SSH
```

RPC `8545`는 public internet에 직접 열지 말고 VPN, security group, firewall로 AWS txgen만 접근하게 제한할 것.

반복 실험 시에는 edge node datadir를 초기화하거나 새 key set을 생성하는 것이 권장됨.

## Reference

이 코드는 원본 [AvivYaish/SpeculativeDoS](https://github.com/AvivYaish/SpeculativeDoS)를 참조하여, in-process txpool 주입 구조를 generator와 edge node RPC 전송 구조로 분리한 것임을 알림.
