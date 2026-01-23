# Docker + Open vSwitch による VLAN 検証手順書（OpenFlow 不使用）

本手順書は **docker-compose + Linux veth + Open vSwitch** を用いて、
**VLAN access / trunk の基本動作を検証する** ための完全手順です。

* OpenFlow は **使用しません**（OVS の L2 機能のみ）
* WSL2 + Docker 環境を前提
* **veth / namespace / OVS の責務分離** が分かる構成

---

## 前提構成（完成形イメージ）

```
[VLAN10]        [VLAN20]
+------+        +------+
|  h1  |        |  h2  |
+--+---+        +--+---+
   |                 |
   | access          | access
   | VLAN10          | VLAN20
   |                 |
   +------veth-------+
            |
        +---+------------------+
        |   Open vSwitch br0   |
        +---+------------------+
            |
           trunk (10,20)
            |
         +--+---+
         |  h3  |
         +------+
```

---

## Step1 : イメージビルドとコンテナ起動

### ビルド

```bash
docker compose build
```

### 起動

```bash
docker compose up -d
```

### 確認

```bash
docker ps
```

すべてのコンテナが起動していることを確認してください。

### OVSブリッジ確認
コンテナ名 OVS に、 内部の論理ブリッジ br0 が存在することを確認する

```bash
docker exec ovs ovs-vsctl show
```

期待結果（br0が既に存在）:

```
Bridge br0
    datapath_type: netdev
    Port br0
        Interface br0
            type: internal
```

**注意**: もし br0 が存在しない場合は、以下のコマンドで手動作成してください:

```bash
docker exec ovs ovs-vsctl add-br br0 -- set Bridge br0 datapath_type=netdev
```

---

## Step2 : veth 作成（OVS コンテナ内）

コンテナ名 OVS の中にある、Linux network namespace の中に veth のペアを作る。
* ペア1：veth-h1 <----> veth-h1-ovs
* ペア2：veth-h2 <----> veth-h2-ovs
* ペア3：veth-h3 <----> veth-h3-ovs

```bash
docker exec ovs ip link add veth-h1 type veth peer name veth-h1-ovs
docker exec ovs ip link add veth-h2 type veth peer name veth-h2-ovs
docker exec ovs ip link add veth-h3 type veth peer name veth-h3-ovs
```

### OVS側のvethをUPにする（重要）

```bash
docker exec ovs ip link set veth-h1-ovs up
docker exec ovs ip link set veth-h2-ovs up
docker exec ovs ip link set veth-h3-ovs up
```

### 確認

```bash
docker exec ovs ip link show | grep veth
```

期待結果: 6つのインターフェース（3つのvethペア）が表示される

---

## Step3 : veth を各コンテナの namespace に移動

### PID 取得

```bash
pid_h1=$(docker inspect -f '{{.State.Pid}}' h1)
pid_h2=$(docker inspect -f '{{.State.Pid}}' h2)
pid_h3=$(docker inspect -f '{{.State.Pid}}' h3)
pid_ovs=$(docker inspect -f '{{.State.Pid}}' ovs)

echo "h1 PID: $pid_h1"
echo "h2 PID: $pid_h2"
echo "h3 PID: $pid_h3"
echo "ovs PID: $pid_ovs"
```

### veth 移動（nsenterを使用）
コンテナ名 OVS の中にある、Linux network namespace の中に veth のペアのうち、
片方を別コンテナのnetwork namespace に移動する。
* ペア1：veth-h1 <----> veth-h1-ovsは、veth-h1 を コンテナ名 h1 に移動する
* ペア2：veth-h2 <----> veth-h2-ovsは、veth-h2 を コンテナ名 h2 に移動する
* ペア3：veth-h3 <----> veth-h3-ovsは、veth-h3 を コンテナ名 h3 に移動する


```bash
sudo nsenter -t $pid_ovs -n ip link set veth-h1 netns $pid_h1
sudo nsenter -t $pid_ovs -n ip link set veth-h2 netns $pid_h2
sudo nsenter -t $pid_ovs -n ip link set veth-h3 netns $pid_h3
```

### 確認

OVS側には -ovs 付きのみ残る:

```bash
docker exec ovs ip link show | grep veth
```

期待結果: `veth-h1-ovs`, `veth-h2-ovs`, `veth-h3-ovs` のみ（3つ）

各コンテナに veth が移動している:

```bash
docker exec h1 ip link show | grep veth
docker exec h2 ip link show | grep veth
docker exec h3 ip link show | grep veth
```

期待結果: それぞれ `veth-h1`, `veth-h2`, `veth-h3` が表示される

### 構造（この時点）

```
[コンテナ名 h1 | veth-h1] <----> [veth-h1-ovs | コンテナ名 ovs]
[コンテナ名 h2 | veth-h2] <----> [veth-h2-ovs | コンテナ名 ovs]
[コンテナ名 h3 | veth-h3] <----> [veth-h3-ovs | コンテナ名 ovs]
```

---

## Step4 : OVS にポート追加（VLAN 設定）

### access ポート（h1: VLAN10, h2: VLAN20）

* コンテナ名 OVS の論理ブリッジ br0 に、veth-h1-ovs を「access VLAN ポート（タグ VLAN 10）」として設定する
* コンテナ名 OVS の論理ブリッジ br0 に、veth-h2-ovs を「access VLAN ポート（タグ VLAN 20）」として設定する


```bash
docker exec ovs ovs-vsctl add-port br0 veth-h1-ovs tag=10
docker exec ovs ovs-vsctl add-port br0 veth-h2-ovs tag=20
```

### trunk ポート（h3: VLAN10,20）
* コンテナ名 OVS の論理ブリッジ br0 に、veth-h3-ovs を トランクポートとして設定する

```bash
docker exec ovs ovs-vsctl add-port br0 veth-h3-ovs trunks=10,20
```

### 確認

```bash
docker exec ovs ovs-vsctl show
```

期待結果:

```
Bridge br0
    datapath_type: netdev
    Port veth-h1-ovs
        tag: 10
        Interface veth-h1-ovs
    Port veth-h2-ovs
        tag: 20
        Interface veth-h2-ovs
    Port veth-h3-ovs
        trunks: [10, 20]
        Interface veth-h3-ovs
    Port br0
        Interface br0
            type: internal
```

---

## Step5 : IP / VLAN IF 設定

### h1（VLAN10 - access）
コンテナ名 h1 の network namespace の中で veth-h1 をアップする
コンテナ名 h1 の network namespace の中で veth-h1 に IPv4 アドレスを割り当てる
```bash
docker exec h1 ip link set veth-h1 up
docker exec h1 ip addr add 10.0.10.1/24 dev veth-h1
```

### h2（VLAN20 - access）
コンテナ名 h2 の network namespace の中で veth-h2 をアップする
コンテナ名 h2 の network namespace の中で veth-h2 に IPv4 アドレスを割り当てる

```bash
docker exec h2 ip link set veth-h2 up
docker exec h2 ip addr add 10.0.20.1/24 dev veth-h2
```

### h3（VLAN10,20 - trunk）
コンテナ名 h3 の network namespace の中で veth-h3 をアップする
コンテナ名 h3 の network namespace の中で veth-h3 を親として、以下をほどこす。
* VLAN サブインターフェース（VLAN ID 10）を作成して、IPv4 アドレスを割り当て、アップする
* VLAN サブインターフェース（VLAN ID 20）を作成して、IPv4 アドレスを割り当て、アップする

```bash
# 物理インターフェースをUP
docker exec h3 ip link set veth-h3 up

# VLAN サブインターフェース作成
docker exec h3 ip link add link veth-h3 name veth-h3.10 type vlan id 10
docker exec h3 ip link add link veth-h3 name veth-h3.20 type vlan id 20

# IPアドレス設定
docker exec h3 ip addr add 10.0.10.3/24 dev veth-h3.10
docker exec h3 ip addr add 10.0.20.3/24 dev veth-h3.20

# サブインターフェースをUP
docker exec h3 ip link set veth-h3.10 up
docker exec h3 ip link set veth-h3.20 up
```

### 確認

```bash
docker exec h1 ip addr show veth-h1
docker exec h2 ip addr show veth-h2
docker exec h3 ip addr show | grep veth-h3
```

期待結果: 各インターフェースに IP アドレスが設定され、state UP になっている

---

## Step6 : 動作確認

### VLAN10 接続確認（h1 → h3）

```bash
docker exec h1 ping -c 4 10.0.10.3
```

**期待結果**: ✅ 成功（同じVLAN10内）

### VLAN20 接続確認（h2 → h3）

```bash
docker exec h2 ping -c 4 10.0.20.3
```

**期待結果**: ✅ 成功（同じVLAN20内）

### VLAN 分離確認（h1 → h2）

```bash
docker exec h1 ping -c 4 10.0.20.1
```

**期待結果**: ❌ 失敗（異なるVLAN間は通信不可）

### VLAN 分離確認（h2 → h1）

```bash
docker exec h2 ping -c 4 10.0.10.1
```

**期待結果**: ❌ 失敗（異なるVLAN間は通信不可）

---

## トラブルシューティング

### pingが通らない場合

**1. インターフェースの状態確認**

```bash
docker exec h1 ip link show veth-h1
docker exec ovs ip link show veth-h1-ovs
```

両方とも `state UP` になっているか確認。DOWN の場合:

```bash
docker exec h1 ip link set veth-h1 up
docker exec ovs ip link set veth-h1-ovs up
```

**2. veth の namespace 移動確認**

```bash
# OVS側に3つだけあるか（-ovs付きのみ）
docker exec ovs ip link show | grep veth

# 各コンテナに1つずつあるか
docker exec h1 ip link show | grep veth
docker exec h2 ip link show | grep veth
docker exec h3 ip link show | grep veth
```

もし OVS 側に6つ（両方）ある場合は、Step3 の namespace 移動が失敗しています。

**3. OVS ポート状態確認**

```bash
docker exec ovs ovs-vsctl show
docker exec ovs ovs-ofctl dump-ports br0
```

**4. OVS フロー確認**

```bash
docker exec ovs ovs-ofctl dump-flows br0
```

NORMAL アクションが設定されているか確認。

**5. パケットキャプチャ**

```bash
# OVS側でパケットが届いているか
docker exec ovs tcpdump -i veth-h1-ovs -n icmp

# 別のターミナルから ping 実行
docker exec h1 ping -c 4 10.0.10.3
```

---

## まとめ（レイヤ対応）

| レイヤ | 実体 | 説明 |
|--------|------|------|
| L2 ケーブル | veth ペア | 仮想的なイーサネットケーブル |
| L2 スイッチ | OVS br0 | VLAN対応の仮想スイッチ |
| VLAN | OVS port tag / trunk | access/trunk ポート設定 |
| L3 | Linux IP / VLAN IF | IPアドレスとVLANサブIF |

---

## クリーンアップ

### 完全削除

```bash
docker compose down
```

### イメージも削除

```bash
docker compose down --rmi all
```

または

```bash
docker rmi $(docker images -q 'sample1_vlan-*')
```

### 確認

```bash
# コンテナが残っていないか
docker ps -a | grep -E 'h1|h2|h3|ovs'

# イメージが残っていないか
docker images | grep sample1_vlan
```

すべて空であれば完全にクリーンアップできています。

---

## 次ステップ（予告）

* OpenFlow で VLAN tag を flow で制御
* ovs-vsctl 設定の完全排除
* Ryu / ONOS 連携

👉 この手順は **そのまま OpenFlow 編の前段** になります。