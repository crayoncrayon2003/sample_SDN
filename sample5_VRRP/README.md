## 前提構成（完成形イメージ）

```
┌──────────┐
│  Client  │ 10.0.0.100/24
│          │ Default GW: 10.0.0.254 (VRRP Virtual IP)
└─────┬────┘
      │
  ────┴─────────────────────────
         OVS Bridge (br0)
  ────┬──────────┬──────────────
      │          │
┌─────┴─────┐  ┌┴──────────┐
│ Router1   │  │ Router2   │
│ (MASTER)  │  │ (BACKUP)  │
│ 10.0.0.1  │  │ 10.0.0.2  │
│ VIP: .254 │  │           │
│ Priority: │  │ Priority: │
│   100     │  │   90      │
└───────────┘  └───────────┘
```


# Step0 : 起動
WSL2ではコンテナ内から直接カーネルモジュールをロードできません。
代わりに、ホストのWSL2カーネルでモジュールをロードする必要があります。必要なモジュールを入れます。
```bash
# WSL2のホスト側で実行
sudo modprobe bonding
sudo modprobe vrf

# 確認
lsmod | grep -E 'bonding|vrf'
```

# Step1 : 起動
```bash
docker compose up -d
docker ps
```


# Step2 : ネットワーク接続の設定
veth 構成

```bash
# OVSにポートを追加
docker exec ovs ovs-vsctl add-port br0 veth-r1 -- set interface veth-r1 type=internal
docker exec ovs ovs-vsctl add-port br0 veth-r2 -- set interface veth-r2 type=internal
docker exec ovs ovs-vsctl add-port br0 veth-client -- set interface veth-client type=internal

# インターフェースを明示的にアップ
docker exec ovs ip link set veth-r1 up
docker exec ovs ip link set veth-r2 up
docker exec ovs ip link set veth-client up

# ホスト上でPIDを取得
PID_OVS=$(docker inspect -f '{{.State.Pid}}' ovs)
PID_ROUTER1=$(docker inspect -f '{{.State.Pid}}' router1)
PID_ROUTER2=$(docker inspect -f '{{.State.Pid}}' router2)
PID_CLIENT=$(docker inspect -f '{{.State.Pid}}' client)

# ホストの/var/run/netnsディレクトリを作成
sudo mkdir -p /var/run/netns

# 各コンテナのネットワーク名前空間をホストに登録
sudo ln -sf /proc/$PID_OVS/ns/net /var/run/netns/ovs
sudo ln -sf /proc/$PID_ROUTER1/ns/net /var/run/netns/router1
sudo ln -sf /proc/$PID_ROUTER2/ns/net /var/run/netns/router2
sudo ln -sf /proc/$PID_CLIENT/ns/net /var/run/netns/client

# ホスト側からvethインターフェースを各コンテナに移動
sudo ip netns exec ovs ip link set veth-r1 netns router1
sudo ip netns exec ovs ip link set veth-r2 netns router2
sudo ip netns exec ovs ip link set veth-client netns client

# ホスト側からインターフェース名を変更
sudo ip netns exec router1 ip link set veth-r1 name eth0
sudo ip netns exec router2 ip link set veth-r2 name eth0
sudo ip netns exec client ip link set veth-client name eth0

# シンボリックリンクのクリーンアップ
sudo rm /var/run/netns/ovs /var/run/netns/router1 /var/run/netns/router2 /var/run/netns/client
```

# Step3 : 動作確認
```bash
# Router1のVRRP状態確認(MASTERになっているはず)
docker exec router1 ip addr show eth0

# Router2のVRRP状態確認(BACKUPになっているはず)
docker exec router2 ip addr show eth0

# クライアントから仮想IPへのPing
docker exec client ping -c 3 10.0.0.254

# クライアントからRouter1へのPing
docker exec client ping -c 3 10.0.0.1

# クライアントからRouter2へのPing
docker exec client ping -c 3 10.0.0.2
```

# Step4 : フェイルオーバーテスト
```bash
# Router1(MASTER)を停止
docker stop router1

# Router2がMASTERに昇格したことを確認
sleep 3
docker exec router2 ip addr show eth0

# クライアントから引き続き疎通可能なことを確認
docker exec client ping -c 3 10.0.0.254
```

# Step5 : Router1を復旧
Step4より前は、コンテナ起動後に eth0を作成する手順にしている。
このため、router1のコンテナをダウン、再起動すると、router1からeth0が消えてしまっている。
再度router1にeth0を付与して復旧する手順にしている。

```bash
# Router1を起動
docker start router1
sleep 3

# 新しいvethインターフェースを作成してRouter1に接続
PID_OVS=$(docker inspect -f '{{.State.Pid}}' ovs)
PID_ROUTER1=$(docker inspect -f '{{.State.Pid}}' router1)

sudo mkdir -p /var/run/netns
sudo ln -sf /proc/$PID_OVS/ns/net /var/run/netns/ovs
sudo ln -sf /proc/$PID_ROUTER1/ns/net /var/run/netns/router1

# 新しいvethインターフェースを作成
docker exec ovs ovs-vsctl add-port br0 veth-r1-new -- set interface veth-r1-new type=internal
docker exec ovs ip link set veth-r1-new up

# Router1に移動してeth0にリネーム
sudo ip netns exec ovs ip link set veth-r1-new netns router1
sudo ip netns exec router1 ip link set veth-r1-new name eth0

# クリーンアップ
sudo rm /var/run/netns/ovs /var/run/netns/router1

# 少し待ってから確認(起動スクリプトが自動的にeth0を設定する)
sleep 5

# Router1がMASTERに復帰したことを確認
docker exec router1 ip addr show eth0

# クライアントから疎通確認
docker exec client ping -c 3 10.0.0.254
```

