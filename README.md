###### tags: `研究`
# opennetvm 安裝 及 用法
> **⚠ 提醒:** 
> 需依照 ovnm -> NF -> pktgen 順序執行
## onvm
先調整環境變數
```bash=
sudo apt-get install build-essential linux-headers-$(uname -r) git bc
sudo apt-get install python3
sudo apt-get install libnuma-dev
sudo apt-get update
git clone https://github.com/sdnfv/openNetVM
cd openNetVM
git checkout master
git submodule sync
git submodule update --init
echo export ONVM_HOME=$(pwd) >> ~/.bashrc
cd dpdk
echo export RTE_SDK=$(pwd) >> ~/.bashrc
echo export RTE_TARGET=x86_64-native-linuxapp-gcc  >> ~/.bashrc
echo export ONVM_NUM_HUGEPAGES=1024 >> ~/.bashrc
```
查詢網卡pci port
```bash=
lspci | grep AQC
```
填入正確的pci port
```bash=
export ONVM_NIC_PCI="xx:xx.x" #填入上面查到的網卡 PCI port
source ~/.bashrc
sudo sh -c "echo 0 > /proc/sys/kernel/randomize_va_space"
cd scripts
./install.sh #會滿久的 完成後確認有無成功綁定網卡在DPDK上
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
```bash=
./onvm/go.sh  -k 1 -n 0xFF0 -s stdout -c -m 0,1,2,3
```
## NF
開啟NF指令(以forward為例) & NF 所在位置
```bash=
cd ~/openNetVM/examples/simple_forward #canlab-worker2
```
simple_forward
```bash=
./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 # 加入參數 (pro)

```
busy_forward
```bash=
./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 -t 20 # 加入參數，就算只想當 simple_forward 也必須加上 -t 0 (pro)

```
SFC_measurement
我自己寫的量測 lantency 和 throughput 的NF，參考 [examples/SFC_measurment](examples/SFC_measurment)  
```bash=
./go.sh 1 -d 2 #從 NF1 傳到 NF2 (basic)

sudo ./go.sh -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -d 2 # 加入參數 (pro)

```
NF_router
別人寫的 NF ，參考 [examples/nf_router](examples/nf_router)  
```bash=
./go.sh 1 -f route.conf #記得改 route.conf 內的內容(加上 IP 繞送規則)
sudo ./go.sh nf_router -l 6 -n 3 -- -m 6 -n 1 -r 1 -s -- -f route.conf  

```
## pktgen
> **⚠ 提醒:** 
> 每次重開都要執行一遍


```bash=
cd ~/openNetVM/scripts #jackkuo-Inspiron-3670
```
先關閉網卡，然後前面onvm設定動作要再做一次
```bash=
sudo ifconfig enp1s0 down
./setup_environment.sh
```

安裝相關位置 及 安裝指令
```bash=
cd ~/openNetVM/tools/Pktgen/pktgen-dpdk #jackkuo-Inspiron-3670
```
```bash=
make
```
設定網卡mac & 查看之檔案位置
```bash=
~/openNetVM/tools/Pktgen/openNetVM-Scripts/pktgen-config.lua #jackkuo-Inspiron-3670
```
```bash=
nano pktgen-config.lua
```
修改成以下範例
```lua=
pktgen.set_mac("0", "src", "3c:7c:3f:4b:76:e9"); --jackkuo-Inspiron-3670
pktgen.set_mac("0", "dst", "4c:ed:fb:92:c4:13"); --canlab-worker2

pktgen.set("all", "count", 100000000000); --可自行調整每次送的封包量，建議設個非常大的數字
pktgen.set("all", "rate", 100); --可自行調整網卡發送速率，建議設 100 開啟程式後再用 set 指令調整
```
更改幾行code (0xff 改 0xf)，程式原本就沒寫好吧，改就對了
```bash=
cd ~/openNetVM/tools/Pktgen/openNetVM-Scripts #jackkuo-Inspiron-3670
```
```bash=
nano run-pktgen.sh
```
```lua=
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
```bash=
cd ~/openNetVM/tools/Pktgen/openNetVM-Scripts #jackkuo-Inspiron-3670
```
```bash=
./run-pktgen.sh 1 #傳到 NF1
```
```bash=
str #開始送封包
stp #停止送封包

set all rate 30 #調整封包 rate 為 30%

seq 0 all 4c:ed:fb:92:c4:13 3c:7c:3f:4b:76:e9 10.11.1.17 10.11.1.16/32 1234 1234 ipv4 udp 0 64 #使用 seq 設定發送不同種類封包

set all seq_cnt 2 #設定seq總數量

script PATH_TO_YOUR_SCRIPT #上面2句 seq 指令也可以使用 script 自動填入(我放在 jackkuo-Inspiron-3670 這台電腦/openNetVM/tools/Pktgen/openNetVM-Scripts 目錄內)

page seq #前往seq page查看

page main #回到main page查看

```


# 俊甫程式使用方法
須掛載俊甫的 .patch
並修改對應的.env，
參考 https://gitlab.com/nthu_canlab/chun-fu_kuo/experiment/-/tree/master/ONVM
```bash
# logging level
LOGGING_LEVEL = "DEBUG"

# save path
SAVE_USER = "jackkuo"


TRACE_TOOL = "/home/jackkuo/bin/onvmtrace"

# openNetVM server init script:
ONVM_SERVER = "$ONVM_HOME/onvm/go.sh -k 1 -n 0xFF0 -s web -c -m 0,1,2,3 -p 8080"
ONVM_PORT = "8080"

# ref: https://github.com/opcm/pcm
PCM_SERVER = "/home/jackkuo/Documents/pcm/pcm-sensor-server.x"

# env variables
ENV = "
ONVM_HOME=/home/jackkuo/openNetVM,
RTE_SDK=/home/jackkuo/openNetVM/dpdk,
RTE_TARGET=x86_64-native-linuxapp-gcc,
ONVM_NUM_HUGEPAGES=1024,
ONVM_NIC_PCI=0000:04:00.0
"

# Trace NF
 TRACE = false
```

## collector.py
```bash=
sudo python3 collector.py tcp/bs_8/fw_3/var_1 -trace
```

## trace2Json.py
```bash=
python3 trace2Json.py tcp/bs_8/fw_3/var_1
```
## pre_process.py
```bash=
python3 pre_process.py tcp/bs_8/fw_3/var_1 #非必要
```
## gen_training_data.py
```bash=
python3 gen_training_data.py
```
