# Kafka sm測項v1.4.4

## Catalog

Req

![](.gitbook/assets/image.png)

Response

![](<.gitbook/assets/image (1).png>)



## Additional

Req

![](<.gitbook/assets/image (2).png>)



Response

![](<.gitbook/assets/image (3).png>)

request body

```
{
    "pseudoId": "12",
    "datacenterCode": "advantech-roy",
    "internalHosts": "10.233.7.167:9092",
    "externalHosts": "10.233.7.167:9092",
    "username": "admin",
    "password": "admin-secret",
    "parameters": "{\"producer_byte_rate\":349525,\"consumer_byte_rate\":699050,\"PARTITIONS\":3,\"REPLICATIONS\":1,\"ZOOKEEPER_PEERS\":\"10.233.56.195:2181\",\"KAFKASHELL_IP\":\"10.233.7.167:8001\"}"
}
```

## Provision

Req

![](<.gitbook/assets/image (5).png>)

Response

![](<.gitbook/assets/image (6).png>)

request body

```
{
    "service_id": "wise-paas-kafka-service-broker",
    "plan_id": "kafka-wise-paas-shared",
    "organization_guid": "kafkaOrg",
    "space_guid": "kafkaSpace",
    "parameters": 
        {
            "datacenterCode": "advantech-roy"
        }
}
```

![](<.gitbook/assets/image (4).png>)

## Binding

Req

![](<.gitbook/assets/image (8).png>)

Response

![](<.gitbook/assets/image (9).png>)



request body

```
{
  "service_id": "wise-paas-kafka-service-broker",
  "plan_id": "kafka-wise-paas-shared",
  "organization_guid": "kafkaOrg",
  "space_guid": "kafkaSpace",
  "parameters":{
  	"topics":[
  	  "roytest1",
      "roytest2"
  	]
  }
}
```

![](<.gitbook/assets/image (7).png>)

![](<.gitbook/assets/image (10).png>)



## Unbind

Req

![](<.gitbook/assets/image (12).png>)

Response

![](<.gitbook/assets/image (13).png>)

![](<.gitbook/assets/image (11).png>)

## Deprovision

Req

![](<.gitbook/assets/image (14).png>)

Response

![](<.gitbook/assets/image (15).png>)

![](<.gitbook/assets/image (16).png>)
