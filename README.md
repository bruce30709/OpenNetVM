###### tags: `研究`
# OpennNetVM 安裝 及 用法
> **⚠ 提醒:** 
> 需依照 ovnm ➡ NF ➡ pktgen 順序執行
## onvm
先調整環境變數
```bash
sudo apt-get install build-essential linux-headers-$(uname -r) git bc
sudo apt-get install python3
sudo apt-get install libnuma-dev
sudo apt-get update
git clone https://github.com/bruce30709/openNetVM
# 需要一次性驗證
cd openNetVM
git submodule sync
git submodule update --init
echo export ONVM_HOME=$(pwd) >> ~/.bashrc
cd dpdk
echo export RTE_SDK=$(pwd) >> ~/.bashrc
echo export RTE_TARGET=x86_64-native-linuxapp-gcc  >> ~/.bashrc
echo export ONVM_NUM_HUGEPAGES=1024 >> ~/.bashrc
```
> **⚠ 提醒:** 
> github 一次性驗證請參考 https://www.astralweb.com.tw/github-how-to-update-the-repository-through-two-stage-verification/  

查詢網卡pci port
```bash
lspci | grep AQC
```
填入正確的pci port
```bash
export ONVM_NIC_PCI="xx:xx.x" #填入上面查到的網卡 PCI port
source ~/.bashrc # 上面一句指令也可以 >> ~/.bashrc ，以後就不用再查找了
sudo sh -c "echo 0 > /proc/sys/kernel/randomize_va_space"
cd scripts
./install.sh #會滿久的，make dpdk，只需要執行一次，完成後確認有無成功綁定網卡在DPDK上
./setup_environment.sh #可檢查是否綁定成功，此行每次重開皆要重新執行

cd ..
cd onvm
make
cd ..
cd examples
make
sudo chmod -R 777 . #如果要跑我的模擬程式，要加入這行
cd ..
```
開啟onvm
```bash
./onvm/go.sh  -k 1 -n 0xFF0 -s stdout -c -m 0,1,2,3
```
## NFs
NF 所在位置
```bash
cd ~/openNetVM/examples/ #canlab-worker2
```
simple_forward
```bash
cd simple_forward

./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 # 加入參數 (pro)

```


busy_forward
俊甫寫的 NF，參考 [busy_forward](examples/busy_forward)
```bash
cd busy_forward

./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 -t 20 # 加入參數，就算只想當 simple_forward 也必須加上 -t 0 (pro)

```
SFC_measurement
我自己寫的量測 lantency 和 throughput 的 NF，參考 [SFC_measurment](examples/SFC_measurment)  
```bash
cd SFC_measurment #當初命名少一個e就將就用 XD

./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 # 加入參數 (pro)

```
NF_router
別人寫的 NF，做封包分流 ，參考 [nf_router](examples/nf_router)  
```bash
cd NF_router

./go.sh 1 -f route.conf #記得改 route.conf 內的內容(加上 IP 繞送規則)

sudo ./go.sh nf_router -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -f route.conf # 加入參數 (pro)  

```
## pktgen
> **⚠ 提醒:** 
> 每次重開都要執行一遍


```bash
cd ~/openNetVM/scripts #packet generator1
```
先關閉網卡，然後前面onvm設定動作要再做一次
```bash
sudo ifconfig enp1s0 down
./setup_environment.sh
```

安裝相關位置 及 安裝指令
```bash
cd ~/openNetVM/tools/Pktgen/pktgen-dpdk #packet generator1
```
```bash
make
```
設定網卡mac & 查看之檔案位置
```bash
cd ~/openNetVM/tools/Pktgen/openNetVM-Scripts/ #packet generator1
```
```bash
nano pktgen-config.lua
```
修改成以下範例
```lua
pktgen.set_mac("0", "src", "AA:BB:CC:DD:EE:FF"); --packet generator1
pktgen.set_mac("0", "dst", "GG:HH:II:JJ:KK:LL"); --packet generator2

pktgen.set("all", "count", 100000000000); --可自行調整每次送的封包量，建議設個非常大的數字
pktgen.set("all", "rate", 100); --可自行調整網卡發送速率，建議設 100 開啟程式後再用 set 指令調整
```
更改幾行code (0xff 改 0xf)，程式原本就沒寫好吧，改就對了
```bash
cd ~/openNetVM/tools/Pktgen/openNetVM-Scripts #packet generator1
```
```bash
nano run-pktgen.sh
```
```lua
# Pktgen has to be started from pktgen-dpdk/
if [ "$PORT_NUM"  -eq "2" ]; then
    (cd "$PKTGEN_HOME" && sudo "$PKTGEN_BUILD" -c 0xf -n 3 -- -p 0x3 "$PORT_MASK" -P -m "[1:2].0, [3:4].1>
elif [ "$PORT_NUM" -eq "1" ]; then
    (cd "$PKTGEN_HOME" && sudo "$PKTGEN_BUILD" -c 0xf -n 3 -- -p 0x1 "$PORT_MASK" -P -m "[1:2].0" -f "$PK>
else
    echo "Helper script only supports 1 or 2 ports"
    exit 0
fi

echo "Pktgen done"

```
開始打封包
```bash
cd ~/openNetVM/tools/Pktgen/openNetVM-Scripts #packet generator1
```
```bash
./run-pktgen.sh 1 #傳到你綁定的portt
```
```bash
str #開始送封包
stp #停止送封包

set all rate 30 #調整封包 rate 為 30%

seq 0 all GG:HH:II:JJ:KK:LL AA:BB:CC:DD:EE:FF 10.11.1.17 10.11.1.16/32 1234 1234 ipv4 udp 0 64 #使用 seq 設定發送不同種類封包

set all seq_cnt 2 #設定seq總數量

script set_seq.lua #上面2句 seq 指令也可以使用 script 自動填入

page seq #前往 seq page 查看

page main #回到 main page 查看

```

## Reference
This project is based on and refers to openNetVM, a high performance NFV platform developed by GW and UCR.
For more details, please visit the official openNetVM repository: https://github.com/sdnfv/openNetVM
