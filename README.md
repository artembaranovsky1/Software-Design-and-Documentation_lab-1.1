# Software-Design-and-Documentation_lab-1.1|||


```mermaid
graph LR
    Client[Web / Mobile Client] -- "1. Send Message" --> API[Backend API]
    Client -- "4. Send ACK (Delivered/Read)" --> API
    
    API --> MS[Message Service]
    
    MS -- "2. Create Message (Status: Sent)" --> DB[(Messages DB)]
    MS -. "5. Update Status (Sent -> Delivered -> Read)" .-> DB
    
    MS --> Queue[Message Queue]
    Queue --> DS[Delivery Service]
    DS --> WS[WebSocket / Push Service]
    
    WS -- "3. Push Notification" --> Client
```

```mermaid
sequenceDiagram
    participant C as Client A
    participant API as Backend API
    participant MS as Message Service
    participant Q as Message Queue
    participant DS as Delivery Service
    participant CB as Client B (Recipient)


    C->>API: POST /send (status: pending)
    API->>MS: processMessage()
    MS->>Q: enqueue delivery task
    API-->>C: 202 Accepted (status: sent)
    
    Q->>DS: trigger delivery
    DS->>CB: push message via WebSocket
    CB-->>DS: ACK (received)
    DS->>MS: updateStatus(delivered)
    MS-->>C: WebSocket: Notify "Delivered"
```


```mermaid
stateDiagram-v2
    [*] --> Sent: Client sends message
    
    Sent --> Delivered: Delivery ACK received\n(System confirm)
    
    Delivered --> Read: Read ACK received\n(User opened)
    
    Read --> [*]

    note right of Sent
        Initial state: 
        Persisted in DB
    end note

    note right of Delivered
        Message reached 
        recipient device
    end note

    note right of Read
        Final state: 
        User viewed content
    end note
```


Status
Accepted

Context
Нам потрібно гарантувати точне відображення статусів повідомлень. Сервер може знати, що він відправив повідомлення, але тільки клієнт може підтвердити, що воно було отримане (delivered) або прочитане (read).

Decision
Who updates status: Клієнтська програма ініціює оновлення, надсилаючи короткі повідомлення-підтвердження (ACK) через API або WebSocket.

Missing Acknowledgements: Якщо сервер не отримує delivered протягом певного часу, повідомлення вважається "в черзі". При наступному підключенні клієнта (B), система проводить синхронізацію і повторно надсилає відсутні ACKs.

Alternatives
Server-only tracking: Сервер ставить статус delivered одразу після відправки в сокет. (Відхилено: не гарантує, що додаток на телефоні реально отримав дані).

Polling: Клієнт постійно запитує, чи є нові статуси. (Відхилено: надто велике навантаження на батарею та мережу).

Consequences
+ Висока точність статусів (як у Telegram/WhatsApp).

- Збільшення кількості дрібних запитів до API (необхідна оптимізація через batching підтверджень).
