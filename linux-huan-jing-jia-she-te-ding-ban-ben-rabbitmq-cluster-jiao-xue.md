# Linux 環境架設特定版本 RabbitMQ cluster 教學

如果沒有限制特定版本的需求, 安裝最新版本的過程較為容易, 參考官方文件即可:

{% embed url="https://www.rabbitmq.com/download.html" %}

這邊主要討論的是如何安裝特定版本的rabbitmq 及其相依的erlanf/OTP 套件, 並且如何讓不同機器之間的 rabbitmq-server 節點可以成立成一個 cluster 架構

## Step 0. 前置步驟

#### - 0.1 確認內核版本

首先先確認一下 kernel 版本,以下例子得知 kernel 為 xenial(16.04)

```
##確認內核資訊
$ sudo lsb_release -a

No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.6 LTS
Release:        16.04
Codename:       xenial
```

#### - 0.2 確認 erlang/OPT 對應版本

由於在rabbitmq 的訊息處理是使用 erlang 語言編譯的, 所以在裝 rabbitmq server 之前,必須要先安裝 erlang。 我們必須先決定要安裝的 rabbitmq-server 版本, 才能知道erlang要對應到哪個版本, 可以從官網查看對應關係。

[https://www.rabbitmq.com/which-erlang.html](https://www.rabbitmq.com/which-erlang.html)

假設我們要裝的 rabbitmq-server 版本為3.7.5, 可以得知對應的erlang/OTP 版本要為20.x 系列

![](<.gitbook/assets/image (17).png>)

## Step 1. 安裝

#### - 1.1 下載 erlang source solution 集合包

參考下方的 erlang deb 列表, 這邊以 erlang 20.3.8.22.1 為例, 需要下載對應我們機器的版本,例如 kernel 是 xenial x86\_64 的話就要下載 xenial 的 amd64 deb

{% embed url="https://packages.erlang-solutions.com/erlang/debian/pool/" %}

![](<.gitbook/assets/image (18).png>)

```
## 下載 erlang solution 包 (xenial)
$ wget https://packages.erlang-solutions.com/erlang/debian/pool/esl-erlang_20.3.8.26-1~ubuntu~xenial_amd64.deb

## 下載 erlang solution 包 (bionic)
$ wget https://packages.erlang-solutions.com/erlang/debian/pool/esl-erlang_20.3.8.26-1~ubuntu~bionic_amd64.deb



```

#### - 1.2 安裝 erlang source solution 集合包

安裝後會看到少了很多相依套件, 在使用 -f 指令去修正後再重裝一次即可

```
## 安裝deb 集合包
$ dpkg -i esl-erlang_20.3.8.26-1~ubuntu~xenial_amd64.deb

## 可以看到少一堆相依賴套件
pkg: dependency problems prevent configuration of esl-erlang:
sl-erlang depends on libwxbase2.8-0 | libwxbase3.0-0 | libwxbase3.0-0v5; however:
Package libwxbase2.8-0 is not installed.
Package libwxbase3.0-0 is not installed.
Package libwxbase3.0-0v5 is not installed.
esl-erlang depends on libwxgtk2.8-0 | libwxgtk3.0-0 | libwxgtk3.0-0v5; however:
Package libwxgtk2.8-0 is not installed.
Package libwxgtk3.0-0 is not installed.
Package libwxgtk3.0-0v5 is not installed.
esl-erlang depends on libsctp1; however:
Package libsctp1 is not installed.

## 修復相依套件
$ sudo apt-get -f install

## 再次執行安裝
$ dpkg -i esl-erlang_20.3.8.26-1~ubuntu~xenial_amd64.deb
```

#### 安裝完以後可以使用 erl 指令確認版本是否正確

```
## 檢查 erlang 版本
$erl -eval '{ok, Version} = file:read_file(filename:join([code:root_dir(), "releases", erlang:system_info(otp_release), "OTP_VERSION"])), io:fwrite(Version), halt().' -noshell
20.3.8.26
```

#### Ps. \[雷坑]網路上有許多範例都是要自己將特定套件版本位址加入 /etc/apt/sources.list, 例如:

```
#假設裝要 22.x 版本 erlang
$ vi /etc/apt/sources.list

#加入:
deb https://dl.bintray.com/rabbitmq-erlang/debianxenial erlang-22.x

$ apt-get update
$ apt-get install erlang=1.22.x-x

...
...
...噴錯

# 會少一堆相依賴套件然後要一個一個自己去找來安裝並且也要版控同樣版本
```

#### 如想要透過該方法再透過 apt-get update 後來安裝的話,或是直接去抓 erlang source code 來 build 的話不是不可以, 但是非常不建議, 因為要自己處理所有相依套件(有數十個)的板控,一開始在這邊搞了非常久, 而且透過 apt get 安裝居然還是官方所寫的安裝方式。

#### - 1.3 下載 rabbitmq server 包

參考下方的 rabbitmq deb 列表, 這邊以 rabbitmq-server 3.7.5-1 為例, 需要下載對應的版本

{% embed url="https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.7.5" %}

![](<.gitbook/assets/image (19).png>)

```
## 下載 rabbitmq-server 集合包
$ wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.5/rabbitmq-server_3.7.5-1_all.deb
```

#### - 1.4 安裝 rabbitmq-server

```
## 安裝 rabbitmq-server
$ dpkg -i rabbitmq-server_3.7.5-1_all.deb

## 修復相依套件
$ sudo apt-get -f install

## 再次執行安裝
$ dpkg -i rabbitmq-server_3.7.5-1_all.deb

## 外掛 manager 插件
$ rabbitmq-plugins enable rabbitmq_management

```

#### 安裝完成後直接去管理頁面看看板號是否正確

![](<.gitbook/assets/image (24).png>)

#### - 1.5 需要理解的 rabbitmq-server檔案位置

rabbitmq-server 安裝完畢後便會自動啟動,後面章節還需要另外配置,這邊需要稍微了解一下基礎檔案的位置資訊:

```
## rabbitmq.conf 配置檔案
$vi cat /etc/rabbitmq/rabbitmq.conf
...
...
## rabbitmq log 查看 log 方便debug
$ tail -f /var/log/rabbitmq/YOUR_DOMAIN_NAME.log

## rabbitmq plugins 放置 ez 檔案的位置
$ ls usr/lib/rabbitmq/lib/rabbitmq_server-YOUR_VERSION/plugins/
```

## Step 2. 配置 RabbitMQ

#### - 2.1 配置 rabbitmq.conf, 細節意義可以參考下面官網說明, 這邊不再贅述每一個屬性所代表的意義直接寫在腳本註解中。注意需要把憑證檔案放到對應資料夾內。

[https://www.rabbitmq.com/configure.html](https://www.rabbitmq.com/configure.html)

```
vi /etc/rabbitmq/rabbitmq.config

# 注意pem檔案的位置要調整
#==================================================================
[
 {kernel,
  [
    {inet_dist_listen_min, 33672},
    {inet_dist_listen_max, 33672}
  ]},
 {rabbit,
  [
    {handshake_timeout, 30000},
    {ssl_handshake_timeout, 30000},
    {collect_statistics_interval,120000},
    {vm_memory_high_watermark,0.66},
    {ssl_listeners, [5671]}, 
    #配置pem檔案位
    {ssl_options, [{cacertfile,"/var/vcap/jobs/rabbitmq-server/etc/cacert.pem"},
                   {certfile,  "/var/vcap/jobs/rabbitmq-server/etc/cert.pem"},
                   {keyfile,   "/var/vcap/jobs/rabbitmq-server/etc/key.pem"}
    ]},
    #配置tcp監聽端口
   {tcp_listen_options, [{backlog,4096},{nodelay,true},{linger,{true,0}},{exit_on_close, false},{sndbuf,32768},{recbuf,32768},{keepalive, true}]}
  ]},
  {rabbitmq_management,
  [
    {rates_mode,basic}
  ]},
  {rabbitmq_mqtt,#配 rabbitmq_mqtt plugin 開放端口
  [
    {tcp_listen_options, [{backlog,4096},{nodelay,true},{linger,{true,0}},{exit_on_close, false},{sndbuf,32768},{recbuf,32768},{keepalive, true}]},
    {ssl_listeners,    [8883]}
  ]},

  {prometheus, [
    {rabbitmq_exporter, [
      {connections_total_enabled, true}
    ]}
  ]},

  {rabbitmq_web_mqtt,
  [
    {ws_path, "/ws"}
  ]},

  {rabbitmq_stomp,
  [
        {tcp_listen_options, [{backlog,4096},{nodelay,true},{linger,{true,0}},{exit_on_close, false},{sndbuf,32768},{recbuf,32768},{keepalive, true}]},
    {ssl_listeners,    [61614]}
  ]}
].
#==================================================================
:wq
```

####  - 2.2 再次重啟 rabbitmq-server

```
## 再次執行安裝, 為了避免 config 沒生效

$ dpkg -r rabbitmq-server

$ dpkg -i rabbitmq-server_3.7.5-1_all.deb


```

#### - 2.3 配置 plugins後, 可以用 list 指令檢查是否成功

```
$ rabbitmq-plugins enable rabbitmq_management rabbitmq_amqp1_0 rabbitmq_auth_backend_cache rabbitmq_auth_backend_http rabbitmq_auth_backend_ldap rabbitmq_auth_mechanism_ssl rabbitmq_consistent_hash_exchange rabbitmq_event_exchange rabbitmq_federation rabbitmq_federation_management rabbitmq_jms_topic_exchange rabbitmq_management rabbitmq_mqtt rabbitmq_shovel rabbitmq_shovel_management rabbitmq_stomp rabbitmq_tracing rabbitmq_web_mqtt rabbitmq_web_stomp rabbitmq_web_stomp_examples

$ rabbitmq-plugins list

ubuntu@rabbitmq21-7902-iaas:~$ sudo -s
root@rabbitmq21-7902-iaas:~# rabbitmq-plugins list
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@rabbitmq21-7902-iaas
 |/
[E*] rabbit_presence_exchange          3.5.1-20150421
[E*] rabbit_udp_exchange               3.5.1-20150421
[E*] rabbitmq_amqp1_0                  3.7.5
[E*] rabbitmq_auth_backend_cache       3.7.5
[E*] rabbitmq_auth_backend_http        3.7.5
[E*] rabbitmq_auth_backend_ldap        3.7.5
[E*] rabbitmq_auth_mechanism_ssl       3.7.5
[E*] rabbitmq_consistent_hash_exchange 3.7.5
[E*] rabbitmq_event_exchange           3.7.5
[E*] rabbitmq_federation               3.7.5
[E*] rabbitmq_federation_management    3.7.5
[E*] rabbitmq_jms_topic_exchange       3.7.5
[E*] rabbitmq_lvc_exchange             20171215-3.6.x
[E*] rabbitmq_management               3.7.5
[e*] rabbitmq_management_agent         3.7.5
[E*] rabbitmq_mqtt                     3.7.5
[  ] rabbitmq_peer_discovery_aws       3.7.5
[  ] rabbitmq_peer_discovery_common    3.7.5
[  ] rabbitmq_peer_discovery_consul    3.7.5
[  ] rabbitmq_peer_discovery_etcd      3.7.5
[  ] rabbitmq_peer_discovery_k8s       3.7.5
[  ] rabbitmq_random_exchange          3.7.5
[  ] rabbitmq_recent_history_exchange  3.7.5
[  ] rabbitmq_sharding                 3.7.5
[E*] rabbitmq_shovel                   3.7.5
[E*] rabbitmq_shovel_management        3.7.5
[E*] rabbitmq_stomp                    3.7.5
[  ] rabbitmq_top                      3.7.5
[E*] rabbitmq_tracing                  3.7.5
[  ] rabbitmq_trust_store              3.7.5
[e*] rabbitmq_web_dispatch             3.7.5
[E*] rabbitmq_web_mqtt                 3.7.5
[  ] rabbitmq_web_mqtt_examples        3.7.5
[E*] rabbitmq_web_stomp                3.7.5
[E*] rabbitmq_web_stomp_examples       3.7.5
```

## Step 3. 配置 RabbitMQ Cluster

#### - 3.1 先確認環境

#### 假設目前造著步驟0\~2, 已經裝好3台 rabbitmq-server 環境, 其名稱及domain name 需要加入各自節點的 /etc/hosts 內如下:

```
# 三台節點都需要同步
$ vim /etc/hosts
127.0.0.1 localhost
10.0.0.20 rabbitmqnode21.rabbitmqcluster rabbitmqnode21 rabbitmq21-7902-iaas rabbit@rabbitmq21-7902-iaas
10.0.0.26 rabbitmqnode22.rabbitmqcluster rabbitmqnode22 rabbitmq22-7905-iaas rabbit@rabbitmq22-7905-iaas
10.0.0.19 rabbitmqnode23.rabbitmqcluster rabbitmqnode23 rabbitmq23-7908-iaas rabbit@rabbitmq23-7908-iaas
```

#### -3.2 同步Erlang Cookie

#### 先啟動要用來充當cluster master node 的 rabbitmq app, 此時會產生 /var/lib/rabbitmq/.erlang.cookie 這份文件，然後再把這份文件copy 到其他cluster node下相對應的目錄下面去。Erlang.cookie 是 erlang 實現分布式的必要文件，erlang 分布式的每個節點要保持相同的.erlang.cookie文件，同時修改完成以後最後需要保證該文件的權限是400。操作如下:

```
# 選定一台當作 Cluster Master Node (假設是root@rabbitmq21-7902-iaas)

$ rabbitmqctl start_app

$ cd /var/lib/rabbitmq/

$ ls -al

total 24
drwxr-xr-x  5 rabbitmq rabbitmq 4096 Oct 28 02:04 .
drwxr-xr-x 48 root     root     4096 Oct 28 10:44 ..
drwxr-x---  3 rabbitmq rabbitmq 4096 Oct 28 02:57 config
-r--------  1 rabbitmq rabbitmq   20 Oct 28 00:00 .erlang.cookie
drwxr-x---  4 rabbitmq rabbitmq 4096 Oct 29 05:34 mnesia
drwxr-x---  2 rabbitmq rabbitmq 4096 Oct 28 02:46 schema

$ chmod 777 .erlang.cookie

$ vi .erlang.cookie

# 將該台的erlang cookie copy 給其他兩台, 
#或是將其他台擔任主節點的 erlang cookie 內容 copy 過來

$ chmod 400 .erlang.cookie

# 改完 cookie 以後要記得把權限改回來,否則會噴錯無法啟動


```

#### -3.3 依序加入 cluster

```
# 確認 cluster 狀態
$ rabbitmqctl cluster_status

# 要加入別人的節點需要先停止該節點app, 要被 join 的節點則必須啟動
$ rabbitmqctl stop_app

# 將該節點加入以 rabbitmq21 為主節點的 cluster中, 主節點
$ rabbitmqctl join_cluster --disc rabbit@rabbitmqnode21.rabbitmqcluster

# 啟動該節點 app
$ rabbitmqctl start_app

# 確認cluster 狀態是否改變
$ rabbitmqctl cluster_status

Cluster status of node rabbit@rabbitmq22-7905-iaas ...
[{nodes,[{disc,['rabbit@rabbitmq21-7902-iaas','rabbit@rabbitmq22-7905-iaas',
                'rabbit@rabbitmq23-7908-iaas']}]},
 {running_nodes,['rabbit@rabbitmq23-7908-iaas','rabbit@rabbitmq21-7902-iaas',
                 'rabbit@rabbitmq22-7905-iaas']},
 {cluster_name,<<"rabbit@rabbitmq21-7902-iaas">>},
 {partitions,[]},
 {alarms,[{'rabbit@rabbitmq23-7908-iaas',[]},
          {'rabbit@rabbitmq21-7902-iaas',[]},
          {'rabbit@rabbitmq22-7905-iaas',[]}]}]

```

#### 可以從管理頁面看到已經部署成功

![](<.gitbook/assets/image (23).png>)

#### 同時可以看到各節點也都有進來

![](<.gitbook/assets/image (25).png>)





