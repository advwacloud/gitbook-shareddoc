# Kafka FAQ

### 2020/05/04

#### Roy

1. log.retention.bytes是指segment size還是partition size, 或是整個kafka cluster的總log size?    \
   因為官方只有寫 "The maximum size of the log before deleting it", 而且爬了文, 有很多種說法
2. 如果kafka是shared的話, persistence.size要調整到多大比較適合,有沒有什麼依據? (預設是1Gi)   \
   我這裡想到的是, 這個size會跟user數量上限, 每個user的topic數量上限, 每個topic的retention.bytes上限, replica大小 有關,是不是要先確定以上幾點, 才有辦法訂出每個pod的persistence.size值?
3. 我這裡測試發現, 當kafka broker的PV因為滿了而crash後, 就沒辦法再重啟成功了, 此時是否只能靠手動清PV去排除這點?
4. kafka RP是不是會根據log.retention.check.interval.ms去定期檢查要不要清除data? ex.預設是五分鐘, 每五分鐘會根據RP策略去檢查並清除data,    \
   \
   我測到一種情境, 是在五分鐘內就把PV寫滿了, 然後kafka broker就CRASH (再也起不來了, 因為pv已經滿了), 為了避免這種情況, 是否需要將log.retention.check.interval.ms調低, 極端一點如2秒檢查一次
5. 是不是實務上, 在建立每個topic時, retention.bytes需要有一個上限? 否則會造成一個topic佔滿所有PV的情況? 
6. kafka pv是否可以動態擴容, 就是pod不重啟的情況, 增加pv size?
7. kafka各個pod再什麼時機會用到ephemeral-storage, 特別是kafka broker的ephemeral-storage limit有到8Gi
