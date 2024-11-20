```plantuml
@startuml
autoactivate on
participant BobB as Bob

Bob -> Alice:同步消息
return yes
Bob ->> Alice:异步消息
Alice --> Bob: 返回消息
@enduml
```

