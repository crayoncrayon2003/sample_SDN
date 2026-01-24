## 前提構成（完成形イメージ）

```
┌─────────────────┐
│  RYU Controller │ OpenFlowコントローラ
└────────┬────────┘
         │ OpenFlow Protocol
 ────────┴──────────────────
         │
    ┌────┴──────┐
    │ OVS Switch│ OpenFlow対応スイッチ
    └──┬───┬────┘
       │   │
 ──────┴───┴───────────
    │        │       │
┌───┴──┐ ┌───┴──┐ ┌──┴───┐
│ Host1│ │Host2 │ │Host3 │
│VLAN10│ │VLAN20│ │Bond  │
└──────┘ └──────┘ └──────┘

```


# Step0 : 起動
WSL2ではコンテナ内から直接カーネルモジュールをロードできません。
代わりに、ホストのWSL2カーネルでモジュールをロードする必要があります。必要なモジュールを入れます。
```bash
# WSL2のホスト側で実行
sudo modprobe bonding
sudo modprobe 8021q

# 確認
lsmod | grep -E 'bonding|8021q'
```

# Step1 : 起動
```bash
docker compose up -d
docker ps
```


# Step2 : Controllerとの接続設定
```bash
PID_OVS=$(docker inspect -f '{{.State.Pid}}' ovs)
PID_CTRL=$(docker inspect -f '{{.State.Pid}}' controller)

# netns準備
sudo mkdir -p /var/run/netns
sudo ln -sf /proc/$PID_OVS/ns/net /var/run/netns/ovs
sudo ln -sf /proc/$PID_CTRL/ns/net /var/run/netns/controller

# ホストでvethペアを作成
sudo ip link add veth-ovs type veth peer name veth-ctrl

# 各コンテナに移動
sudo ip link set veth-ovs netns ovs
sudo ip link set veth-ctrl netns controller

# OVS側の設定（docker execを使用）
sudo ip netns exec ovs ip link set veth-ovs up
docker exec ovs ovs-vsctl add-port br0 veth-ovs
docker exec ovs ip addr add 10.0.0.2/24 dev br0
docker exec ovs ip link set br0 up

# Controller側の設定
sudo ip netns exec controller ip link set veth-ctrl name eth0
sudo ip netns exec controller ip addr add 10.0.0.1/24 dev eth0
sudo ip netns exec controller ip link set eth0 up

# OpenFlowコントローラを指定
docker exec ovs ovs-vsctl set-controller br0 tcp:10.0.0.1:6653

# netnsクリーンアップ
sudo rm -f /var/run/netns/ovs /var/run/netns/controller

# 接続確認
echo "Controller-OVS接続確認..."
docker exec controller ping -c 3 10.0.0.2

# OpenFlow接続確認
sleep 3
docker logs controller | tail -20
docker exec ovs ovs-vsctl show
```

# Step3 : ホストの接続設定
```bash
# OVSでvethペアを作成
docker exec ovs ip link add veth-h1 type veth peer name veth-h1-ovs
docker exec ovs ip link add veth-h2 type veth peer name veth-h2-ovs
docker exec ovs ip link add veth-h3-1 type veth peer name veth-h3-1-ovs
docker exec ovs ip link add veth-h3-2 type veth peer name veth-h3-2-ovs
docker exec ovs ip link add veth-r1 type veth peer name veth-r1-ovs
docker exec ovs ip link add veth-r2 type veth peer name veth-r2-ovs

# OVS側をブリッジに追加
docker exec ovs ovs-vsctl add-port br0 veth-h1-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h2-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h3-1-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h3-2-ovs
docker exec ovs ovs-vsctl add-port br0 veth-r1-ovs
docker exec ovs ovs-vsctl add-port br0 veth-r2-ovs

# OVS側をアップ
docker exec ovs ip link set veth-h1-ovs up
docker exec ovs ip link set veth-h2-ovs up
docker exec ovs ip link set veth-h3-1-ovs up
docker exec ovs ip link set veth-h3-2-ovs up
docker exec ovs ip link set veth-r1-ovs up
docker exec ovs ip link set veth-r2-ovs up

# PID取得
PID_OVS=$(docker inspect -f '{{.State.Pid}}' ovs)
PID_HOST1=$(docker inspect -f '{{.State.Pid}}' host1)
PID_HOST2=$(docker inspect -f '{{.State.Pid}}' host2)
PID_HOST3=$(docker inspect -f '{{.State.Pid}}' host3)
PID_ROUTER1=$(docker inspect -f '{{.State.Pid}}' router1)
PID_ROUTER2=$(docker inspect -f '{{.State.Pid}}' router2)

sudo mkdir -p /var/run/netns
sudo ln -sf /proc/$PID_OVS/ns/net /var/run/netns/ovs
sudo ln -sf /proc/$PID_HOST1/ns/net /var/run/netns/host1
sudo ln -sf /proc/$PID_HOST2/ns/net /var/run/netns/host2
sudo ln -sf /proc/$PID_HOST3/ns/net /var/run/netns/host3
sudo ln -sf /proc/$PID_ROUTER1/ns/net /var/run/netns/router1
sudo ln -sf /proc/$PID_ROUTER2/ns/net /var/run/netns/router2

# インターフェースを各コンテナに移動
sudo ip netns exec ovs ip link set veth-h1 netns host1
sudo ip netns exec ovs ip link set veth-h2 netns host2
sudo ip netns exec ovs ip link set veth-h3-1 netns host3
sudo ip netns exec ovs ip link set veth-h3-2 netns host3
sudo ip netns exec ovs ip link set veth-r1 netns router1
sudo ip netns exec ovs ip link set veth-r2 netns router2

# インターフェース名を変更
sudo ip netns exec host1 ip link set veth-h1 name eth0
sudo ip netns exec host2 ip link set veth-h2 name eth0
sudo ip netns exec host3 ip link set veth-h3-1 name eth0
sudo ip netns exec host3 ip link set veth-h3-2 name eth1
sudo ip netns exec router1 ip link set veth-r1 name eth0
sudo ip netns exec router2 ip link set veth-r2 name eth0

sudo rm -f /var/run/netns/ovs /var/run/netns/host1 /var/run/netns/host2 /var/run/netns/host3 /var/run/netns/router1 /var/run/netns/router2

# ホスト側で設定を適用（各コンテナのスタートアップスクリプトを手動実行）
docker exec -d host1 /start.sh
docker exec -d host2 /start.sh
docker exec -d host3 /start.sh
docker exec -d router1 /start.sh
docker exec -d router2 /start.sh

sleep 10

# 確認
docker exec host1 ip addr show
docker exec host2 ip addr show
docker exec host3 ip addr show
```

# Step4 : OpenFlow動作確認
```bash
# OpenFlowコントローラの接続確認
docker exec ovs ovs-vsctl show

# OpenFlowフロー確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# Controllerログ確認（最新20行）
docker logs controller | tail -20
```

# Step5 : VLAN動作確認
```bash
# まず、ポート番号を確認
docker exec ovs ovs-ofctl -O OpenFlow13 show br0

# 現在のフローを確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# VLAN 10とVLAN 20用のフローを追加
# ポート2がHost1、ポート3がHost2であることを前提
docker exec ovs ovs-ofctl -O OpenFlow13 add-flow br0 "priority=100,dl_vlan=10,actions=output:2"
docker exec ovs ovs-ofctl -O OpenFlow13 add-flow br0 "priority=100,dl_vlan=20,actions=output:3"

# フローが追加されたか確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# Host1からHost2へVLANタグ付きパケット送信テスト
# （同一VLAN内での通信テスト）
docker exec host1 ping -I eth0.10 -c 3 192.168.10.1

# tcpdumpでVLANタグ付きパケットをキャプチャ
# 別のターミナルで実行するか、バックグラウンドで実行
docker exec ovs timeout 10 tcpdump -i veth-h1-ovs -e -n vlan &

# Host2からもパケット送信
docker exec host2 ping -I eth0.20 -c 3 192.168.20.1

# パケット統計を確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# コントローラログで学習されたMACアドレスを確認
docker logs controller | grep "packet in" | tail -20
```

# Step6 :  リンクアグリゲーション確認
```bash
# Host3のボンディング状態確認
docker exec host3 cat /proc/net/bonding/bond0

# ボンディングインターフェースの詳細確認
docker exec host3 ip link show bond0
docker exec host3 ip addr show bond0

# OVSでLACPを有効化
# まず、どのポートがHost3に接続されているか確認
docker exec ovs ovs-vsctl show

# Host3用のポート（veth-h3-1-ovs と veth-h3-2-ovs）にLACPを有効化
docker exec ovs ovs-vsctl set port veth-h3-1-ovs lacp=active
docker exec ovs ovs-vsctl set port veth-h3-2-ovs lacp=active

# LACP設定の確認
docker exec ovs ovs-vsctl list port veth-h3-1-ovs | grep lacp
docker exec ovs ovs-vsctl list port veth-h3-2-ovs | grep lacp

# OVS側でLACPパケットをキャプチャ（10秒間）
docker exec ovs timeout 10 tcpdump -i veth-h3-1-ovs -e -n ether proto 0x8809 &

# Host3から通信を発生させる
docker exec host3 ping -I bond0 -c 5 192.168.30.254

# ボンディング統計情報を確認
docker exec host3 cat /proc/net/bonding/bond0

# OpenFlowでLACPパケットの統計を確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0 | grep -i lacp

# コントローラログでLACPパケットを確認
docker logs controller | grep "01:80:c2:00:00:02" | tail -10
```

# Step7 :  VRRP動作確認
```bash
# Router1の状態確認（MASTERでVIP保持）
docker exec router1 ip addr show eth0

# Router2の状態確認（BACKUP）
docker exec router2 ip addr show eth0

# VRRPの仮想IP（VIP）が見えるか確認
# Router1に 192.168.100.254 が存在するはず
docker exec router1 ip addr show eth0 | grep 192.168.100.254

# Router2には 192.168.100.254 が存在しないはず
docker exec router2 ip addr show eth0 | grep 192.168.100.254

# Keepalivedのログを確認
docker logs router1 | grep -i vrrp | tail -20
docker logs router2 | grep -i vrrp | tail -20

# VRRPパケットをキャプチャ（10秒間、別ターミナル推奨）
docker exec ovs timeout 10 tcpdump -i veth-r1-ovs -n proto 112 &

# Router1を停止してフェイルオーバーテスト
echo "Router1を停止します..."
docker stop router1

# 少し待つ
sleep 5

# Router2がMASTERに昇格してVIPを持つか確認
echo "Router2の状態確認（MASTERに昇格してVIPを保持するはず）"
docker exec router2 ip addr show eth0
docker exec router2 ip addr show eth0 | grep 192.168.100.254

# Router2のログでMASTER昇格を確認
docker logs router2 | grep -i "entering master state" | tail -5

# Router1を再起動してフェイルバック確認
echo "Router1を再起動します..."
docker start router1
sleep 10

# Router1がMASTERに戻るか確認
docker exec router1 ip addr show eth0 | grep 192.168.100.254

# Router2がBACKUPに戻るか確認
docker exec router2 ip addr show eth0 | grep 192.168.100.254

# 両方のログを確認
docker logs router1 | grep -i "entering master state" | tail -3
docker logs router2 | grep -i "entering backup state" | tail -3
```


# Step8 :  OpenFlowでVRRPトラフィック制御
```bash
# 現在のフローエントリを確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# VRRPパケット（IPプロトコル番号112）を優先的に転送するフローを追加
docker exec ovs ovs-ofctl -O OpenFlow13 add-flow br0 "priority=200,ip,nw_proto=112,actions=normal"

# フローが追加されたか確認
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0 | grep "nw_proto=112"

# VRRPパケットをキャプチャ（20秒間）
echo "VRRPパケットをキャプチャ中..."
docker exec ovs timeout 20 tcpdump -i br0 -n proto 112 -v

# フロー統計を確認（VRRPパケットがマッチしているか）
echo "フロー統計確認..."
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0 | grep "nw_proto=112"

# コントローラログでVRRPパケット処理を確認
docker logs controller | grep -E "(224\.0\.0\.18|01:00:5e:00:00:12)" | tail -10

# 全体のフロー統計サマリー
echo "全フロー統計サマリー..."
docker exec ovs ovs-ofctl -O OpenFlow13 dump-flows br0

# OVS全体の統計情報
docker exec ovs ovs-ofctl -O OpenFlow13 dump-aggregate br0

# 最終確認：VRRPが正常に動作しているか
echo "Router1 VIP確認..."
docker exec router1 ip addr show eth0 | grep 192.168.100.254 && echo "Router1: MASTER (VIP保持)" || echo "Router1: BACKUP"

echo "Router2 VIP確認..."
docker exec router2 ip addr show eth0 | grep 192.168.100.254 && echo "Router2: MASTER (VIP保持)" || echo "Router2: BACKUP"
```



# Step9 :  環境のクリーンアップ
```bash
docker-compose down
sudo rm -f /var/run/netns/*
```