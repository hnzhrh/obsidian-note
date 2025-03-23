```mermaid

sequenceDiagram
participant Alice participant Bob

Bob ->> Alice: 同步消息
Bob -) Alice: 异步消息
Alice -->> Bob: 返回消息

```

