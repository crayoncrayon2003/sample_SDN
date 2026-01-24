## 前提構成（完成形イメージ）
```
[コンテナ名 h1 | veth-h1 ] ── [ veth-h1-ovs1                 veth-h4-ovs1] ── [ veth-h4 | コンテナ名 h4]
                                [               コンテナ名 OVS             ]
[コンテナ名 h2 | veth-h2 ] ── [ veth-h2-ovs1                 veth-h3-ovs1] ── [ veth-h3 | コンテナ名 h3]
```

* h1 と OVS の間は、 veth-h1 と veth-h1-ovs の1本で接続
* h2 と OVS の間は、 veth-h2 と veth-h2-ovs の1本で接続
* h3 と OVS の間は、 veth-h3 と veth-h3-ovs の1本で接続
* h4 と OVS の間は、 veth-h4 と veth-h4-ovs の1本で接続

* OVSには、VRF-A と VRF-B
* VRF-Aには、veth-h1-ovs と veth-h2-ovs が所属
* VRF-Bには、veth-h3-ovs と veth-h4-ovs が所属

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
* ペア3：veth-h3 <----> veth-h3-ovs
* ペア4：veth-h3 <----> veth-h4-ovs
```bash
docker exec ovs ip link add veth-h1 type veth peer name veth-h1-ovs
docker exec ovs ip link add veth-h2 type veth peer name veth-h2-ovs
docker exec ovs ip link add veth-h3 type veth peer name veth-h3-ovs
docker exec ovs ip link add veth-h4 type veth peer name veth-h4-ovs
```

# Step3 : OVS側のvethをUPにする
```bash
docker exec ovs ip link set veth-h1-ovs up
docker exec ovs ip link set veth-h2-ovs up
docker exec ovs ip link set veth-h3-ovs up
docker exec ovs ip link set veth-h4-ovs up
```

# Step4 : veth を各コンテナへ移動
## PID 取得
```bash
pid_ovs=$(docker inspect -f '{{.State.Pid}}' ovs)
pid_h1=$(docker inspect -f '{{.State.Pid}}' h1)
pid_h2=$(docker inspect -f '{{.State.Pid}}' h2)
pid_h3=$(docker inspect -f '{{.State.Pid}}' h3)
pid_h4=$(docker inspect -f '{{.State.Pid}}' h4)
echo "h1 PID: $pid_h1"
echo "h2 PID: $pid_h2"
echo "h3 PID: $pid_h3"
echo "h4 PID: $pid_h4"
echo "ovs PID: $pid_ovs"
```

## 移動
```bash
sudo nsenter -t $pid_ovs -n ip link set veth-h1 netns $pid_h1
sudo nsenter -t $pid_ovs -n ip link set veth-h2 netns $pid_h2
sudo nsenter -t $pid_ovs -n ip link set veth-h3 netns $pid_h3
sudo nsenter -t $pid_ovs -n ip link set veth-h4 netns $pid_h4
```

# Step5 : VRF作成（OVS側）
VRF作成
```bash
docker exec ovs ip link add vrf-A type vrf table 100
docker exec ovs ip link add vrf-B type vrf table 200
docker exec ovs ip link set vrf-A up
docker exec ovs ip link set vrf-B up
```

ポート割当
```bash
docker exec ovs ip link set veth-h1-ovs master vrf-A
docker exec ovs ip link set veth-h2-ovs master vrf-A
docker exec ovs ip link set veth-h3-ovs master vrf-B
docker exec ovs ip link set veth-h4-ovs master vrf-B
```

OVS ブリッジにポート追加
```bash
docker exec ovs ovs-vsctl add-port br0 veth-h1-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h2-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h3-ovs
docker exec ovs ovs-vsctl add-port br0 veth-h4-ovs
```

# Step6 : 各ホスト IP 設定
```bash
# VRF-A
docker exec h1 ip link set veth-h1 up
docker exec h1 ip addr add 10.0.0.1/24 dev veth-h1

docker exec h2 ip link set veth-h2 up
docker exec h2 ip addr add 10.0.0.2/24 dev veth-h2

# VRF-B
docker exec h3 ip link set veth-h3 up
docker exec h3 ip addr add 10.0.1.1/24 dev veth-h3

docker exec h4 ip link set veth-h4 up
docker exec h4 ip addr add 10.0.1.2/24 dev veth-h4
```


# Step7 : 通信確認（ VRF-A <-> VRF-A ）
```bash
docker exec h1 ping -c 3 10.0.0.2
docker exec h2 ping -c 3 10.0.0.1
```

✅ 成功すること

# Step8 : 通信確認（ VRF-B <-> VRF-B ）
```bash
docker exec h3 ping -c 3 10.0.1.2
docker exec h4 ping -c 3 10.0.1.1
```

✅ 通信が継続すること

# Step9 : 通信確認（ VRF-A <-> VRF-B ）
```bash
docker exec h1 ping -c 3 10.0.1.1
```

❎ 通信が出来ないこと

