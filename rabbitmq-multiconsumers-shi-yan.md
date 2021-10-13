# RabbitMQ Multi-Consumers 實驗

### 測試情境:

\-建立一個 ExChange, 型態為 fanout 模式, 即廣播模式,並將該 exchange 綁定至10000個 queue

\-建立一個 producer 連線, 持續每秒對該 exchange 發送 1 資料筆, 相當於每秒對這10000個 queue同時發送1 1筆資料

\-建10000個 cosumer 連線, 各自消費這10000個 queue, 每一個 Channel 只能給一個Consumer消費,因此也會需要 10000 個 Channel

以這樣的方式同時收發放置約24小時, 觀察所有 10000 個 consumer 其消費總筆數是否正常,以及是否有遺漏資料

### 控制變因及資源配置:

```
RabbitMQ 版本:3.7.5 

Erlang 版本:20.3.8.3

節點數:測試單節點

VM 規格: 
    CPU: 4 cores (Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz)
    Memory: 8gb

測試情境:
    Connection 數量: 10001 (10000 consumer + 1 producer)
    Channel 數量:10001 10000 (10000 consumer + 1 producer)
    Queues 數量: 10000
    Consumer 數量: 10000

RabbitMQ 相關配置:
    Consumer 是否使用 AutoAck: 是
    Exchange type: fanout
    Exchange durable: false
    Memory High Watermark: 0.9x
    Memory Calculation Strategy: rss
    是否使用 Mirror 鏡像同步訊息: 否

訊息內容:
    傳送訊息內容:  "test message"+資料筆數
    傳送頻率:   所有10000 個 queue 每秒會有1筆訊息傳入
```

![](<.gitbook/assets/image (34).png>)

### 測試方式:

使用 nodejs 程式語言分別撰寫 amqp publisher 以及consumer, publisher 會固定每秒對 exchange 傳送訊息, 此時會廣播該次訊息同步到 10000 個 queue 中, 再透過 consumer 程式(起5個, 各負責2000個連線)去接收資料, 並印出各個 queue 消費的總筆數, 算出每一個 queue 的消費次數是否跟發送端次數一致, 同時也透過rabbitmq managerment 察看訊息堆積狀態。

![](<.gitbook/assets/image (44).png>)

####

### 測試結果概述:

放置約快22小時, 其間10000個 consumer 在消費訊息時沒有任何遺漏, 總筆數也正確, 也無任何訊息堆積情況, 

資源使用方面記憶體穩最終定維持在約 6.7 gb\~7.0 gb之間, cpu穩定在 169% \~ 202%之間起伏

### 測試紀錄:(繼續放置中)

| Producer 生產資料次數 | Consumer 消費資料總數 | 測試開始時間            | 測試結束時間           |
| --------------- | --------------- | ----------------- | ---------------- |
| 77861           | 778610000       | 約2020/12/10:19:32 | 2020/12/11:17:12 |
|                 |                 |                   |                  |

### 訊息消費概況:

![](<.gitbook/assets/image (35).png>)

### Queue狀況(擷取部分):

每秒接收傳送資料量差不多(deliver/incoming )

![](<.gitbook/assets/image (43).png>)

### 節點記憶體以及磁碟用量:

![](<.gitbook/assets/image (45).png>)

### 節點記憶體詳細用量:

![](<.gitbook/assets/image (46).png>)

![](<.gitbook/assets/image (40).png>)

### 虛擬機器內部資源用量:

![](<.gitbook/assets/image (42).png>)

#### (附錄.1) Producer 測試程序

```
//--producer 模擬程序--

const amqp = require('amqplib');
const CONNECT_URL = 'amqps://user01:123456@61.219.26.62:5671/vhost01';
const EXCHANGE_NAME = 'ex';
const EXCHANGE_TYPE = 'fanout';
const EXCHANGE_OPTION = { durable: false };

const MAX_QUEUES = 10000;
let data_count = 0;

process.env.NODE_TLS_REJECT_UNAUTHORIZED = 0;

let queues = [];

function generateQueue(count){
    for(var i = 1; i <= count ; i++){
        queues.push('queue_'+i);
    }
}

function deleteQueues(channel){
    for (let index = 0; index < queues.length; index++) {
        queue = queues[index];
        channel.deleteQueue(queue, {})
    }
}

function sendData(channel){
    
    data_count ++;
    let message = 'test message' + data_count;

    console.log(message);

    channel.publish(EXCHANGE_NAME, '', new Buffer(message));
}

async function main() {

    generateQueue(MAX_QUEUES);
    
    const conn = await amqp.connect(CONNECT_URL);
    const channel = await conn.createChannel();

    //deleteQueues(channel);  //重製測試環境用,刪除所有 queue
    
    for (let index = 0; index < queues.length; index++) {

        queue = queues[index];  
        await channel.assertQueue(queue, {durable: false});
        console.log('assertQueue...' + queue);

        await channel.assertExchange(EXCHANGE_NAME, EXCHANGE_TYPE, EXCHANGE_OPTION);
        console.log('assertExchange...' + queue);

        await channel.bindQueue(queue, EXCHANGE_NAME, '');
        console.log('bindQueue...' + queue);
    }
    setInterval(() => {
       sendData(channel);
    }, 1000);
}

main();

```

#### (附錄.2) Consumer 測試程序

```
//--consumer 模擬程序--



const amqp = require('amqplib');
const CONNECT_URL = 'amqps://user01:123456@61.219.26.62:5671/vhost01';  // channelMax=2047 0~2047  max channel number: 2048
const EXCHANGE_NAME = 'ex';
const EXCHANGE_TYPE = 'fanout';
const EXCHANGE_OPTION = { durable: false };

//const CHANNEL_LIMIT = 2047; // 0~2047
const START_INDEX = 1;//2001, 4001, 6001, 8001
const END_INDEX = 2000; //4000,6000,8000,10000
const MAX_COMSUMER = 2000;
let data_count = 0;

process.env.NODE_TLS_REJECT_UNAUTHORIZED = 0;

let queues = [];
let channels = [];
let connections = [];

function generateQueue(count){
    for(var i = START_INDEX; i <= END_INDEX ; i++){
        queues.push('queue_'+i);
    }
}

let total_count = 0;
let each_count = 0;

async function main() {

    generateQueue(MAX_COMSUMER);
    
    for(let index =0; index< (MAX_COMSUMER); index++){  // per connection has 2048 channels
        let connection = await amqp.connect(CONNECT_URL);
        //console.log('creating connection...' + index);
        connections.push(connection);    
        
        let channel = await connections[index].createChannel();
        console.log('pushing channel...' + index);
        channels.push(channel) 
    }

    for (let index = 0; index < channels.length; index++) { 
        await channels[index].consume(queues[index], function(msg) {

            total_count ++;
            each_count = total_count/MAX_COMSUMER;

            let tmpResult = {
                total: total_count,
                each: each_count,
                current_consumer: Object.keys(channels[index].consumers),
                current_msg_content: msg.content.toString()
            }

            if(total_count%MAX_COMSUMER == 0){
                console.log('total_count get: ' + total_count + ' ' + total_count/MAX_COMSUMER);
            }
            //console.log(tmpResult);
            //console.log(" [x] %s %s:'%s'", Object.keys(channels[index].consumers), msg.fields.routingKey, msg.content.toString());
        }, {noAck: true});
    }
}



main();


```
