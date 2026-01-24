## 前提構成（完成形イメージ）
```
[コンテナ名 h1 | veth-h1 ] ── [ veth-h1-ovs1                                           ]
                             [               コンテナ名 OVS  veth-h3-ovs1/veth-h3-ovs2 ] ── [ veth-h3-1 / veth-h3-2 | コンテナ名 h3]
[コンテナ名 h2 | veth-h2 ] ── [ veth-h2-ovs1                                           ]
```

* h1 と OVS の間は、 veth-h1 と veth-h1-ovs の1本で接続
* h2 と OVS の間は、 veth-h2 と veth-h2-ovs の1本で接続
* h3 と OVS の間は、 次の2本をリンクアグリゲーションで接続
 * veth-h3-1とveth-h3-ovs1
 * veth-h3-2とveth-h3-ovs2

# Step0 : 起動
WSL2ではコンテナ内から直接カーネルモジュールをロードできません。
代わりに、ホストのWSL2カーネルでモジュールをロードする必要があります。必要なモジュールを入れます。
```bash
# WSL2のホスト側で実行
sudo modprobe bonding

# 確認
lsmod | grep bonding
```

# Step1 : 起動
```bash
docker compose up -d
docker ps
```

OVS ブリッジ確認：
```bash
docker exec ovs ovs-vsctl show
```

期待結果（br0が既に存在）:

```bash
Bridge br0
    datapath_type: netdev
    Port br0
        Interface br0
            type: internal
```

# Step2 : veth 作成（OVS namespace）
veth 構成

コンテナ名 OVS の中にある、Linux network namespace の中に veth のペアを作る。
* ペア1：veth-h1 <----> veth-h1-ovs
* ペア2：veth-h2 <----> veth-h2-ovs
* ペア3：veth-h3-1 <----> veth-h3-ovs1
* ペア4：veth-h3-2 <----> veth-h3-ovs2
```bash
docker exec ovs ip link add veth-h1 type veth peer name veth-h1-ovs
docker exec ovs ip link add veth-h2 type veth peer name veth-h2-ovs
docker exec ovs ip link add veth-h3-1 type veth peer name veth-h3-ovs1
docker exec ovs ip link add veth-h3-2 type veth peer name veth-h3-ovs2
```

# Step3 : OVS側のvethをUPにする
```bash
docker exec ovs ip link set veth-h1-ovs up
docker exec ovs ip link set veth-h2-ovs up
docker exec ovs ip link set veth-h3-ovs1 up
docker exec ovs ip link set veth-h3-ovs2 up
```

# Step4 : veth を各コンテナへ移動
## PID 取得
```bash
pid_ovs=$(docker inspect -f '{{.State.Pid}}' ovs)
pid_h1=$(docker inspect -f '{{.State.Pid}}' h1)
pid_h2=$(docker inspect -f '{{.State.Pid}}' h2)
pid_h3=$(docker inspect -f '{{.State.Pid}}' h3)
echo "h1 PID: $pid_h1"
echo "h2 PID: $pid_h2"
echo "h3 PID: $pid_h3"
echo "ovs PID: $pid_ovs"
```

## 移動
```bash
sudo nsenter -t $pid_ovs -n ip link set veth-h1 netns $pid_h1
sudo nsenter -t $pid_ovs -n ip link set veth-h2 netns $pid_h2

sudo nsenter -t $pid_ovs -n ip link set veth-h3-1 netns $pid_h3
sudo nsenter -t $pid_ovs -n ip link set veth-h3-2 netns $pid_h3
```

# Step5 : OVS 側 bond 作成（LACPなし）
```bash
docker exec ovs ovs-vsctl add-bond br0 bond0 veth-h3-ovs1 veth-h3-ovs2 bond_mode=balance-slb lacp=active

docker exec ovs ovs-vsctl add-port br0 veth-h1-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h2-ovs
```

確認：
```bash
docker exec ovs ovs-vsctl show
```

# Step6 : h3 側 Linux bond 作成
```bash
# h1とh2の設定
docker exec h1 ip link set veth-h1 up
docker exec h1 ip addr add 10.0.0.1/24 dev veth-h1

docker exec h2 ip link set veth-h2 up
docker exec h2 ip addr add 10.0.0.2/24 dev veth-h2

# h3側: インターフェースをdown
docker exec h3 ip link set veth-h3-1 down
docker exec h3 ip link set veth-h3-2 down

# bondを作成(fail_over_macを設定)
docker exec h3 ip link add bond0 type bond mode 802.3ad miimon 100 lacp_rate 1

# スレーブを追加
docker exec h3 ip link set veth-h3-1 master bond0
docker exec h3 ip link set veth-h3-2 master bond0

# 起動
docker exec h3 ip link set bond0 up

# IPアドレス設定
docker exec h3 ip addr add 10.0.0.3/24 dev bond0
```



確認：
```bash
# h3側のbond状態確認
docker exec h3 cat /proc/net/bonding/bond0

# OVS側のbond状態確認
docker exec ovs ovs-appctl bond/show bond0
```

# Step7 : 通信確認（ケース1）
```bash
docker exec h1 ping -c 3 10.0.0.3
docker exec h2 ping -c 3 10.0.0.3
```

✅ 成功すること

# Step8 : 片系断テスト（ケース2）
```bash
docker exec ovs ip link set veth-h3-ovs1 down

docker exec h1 ping -c 3 10.0.0.3
docker exec h2 ping -c 3 10.0.0.3
```

✅ 通信が継続すること

復旧：
```bash
docker exec ovs ip link set veth-h3-ovs1 up
```