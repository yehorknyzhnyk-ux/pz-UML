# UML Моделювання системи сповіщень

## 1. Use Case Diagram

```mermaid
graph TD
    Server["Сервер (Об'єкт)"] --- UC1((Виявлення недоступності))
    Admin["Адміністратор (Клієнт)"] --- UC4((Відтворення повідомлення))
    Asterisk["Asterisk (SIP-сервер)"] --- UC3((Встановлення SIP-з'єднання))

    subgraph "Система моніторингу та зв'язку"
        UC1 --> UC2((Надсилання запиту API))
        UC2 --> Asterisk
        UC3 --> UC4
    end
```
###
```mermaid
    sequenceDiagram
    autonumber
    participant S as Server (Down)
    participant Z as Zabbix Server
    participant A as Asterisk (FreePBX)
    participant P as IP-Phone (Client)

    S-X S: Сервіс зупинено
    Z->>Z: Trigger: High Severity
    Note over Z,A: Виклик API (AMI Originate)
    Z->>A: POST /api/v1/make_call (Ext: 101)

    A->>P: SIP INVITE (Calling...)
    P-->>A: 180 Ringing
    P-->>A: 200 OK (Слухавку піднято)

    Note right of A: Встановлення медіа-потоку
    A->>P: RTP Stream (Audio: "Alert! Server 1 is offline")

    P-->>A: SIP BYE (Клієнт поклав слухавку)
    A-->>Z: Call Result: Success
```
###
```mermaid
    stateDiagram-v2
    [*] --> Monitoring: Zabbix активний
    Monitoring --> IncidentDetected: Сервер не відповідає

    IncidentDetected --> SendApiRequest: Формування API запиту

    state "Процес виклику" as CallProcess {
        [*] --> InitiatingCall
        InitiatingCall --> CallRinging
        CallRinging --> Connected: Клієнт відповів
        CallRinging --> NoAnswer: Тайм-аут (30 сек)
    }

    Connected --> PlayAudio: Програти запис (RTP)
    PlayAudio --> [*]

    NoAnswer --> RetryLogic
    state RetryLogic <<choice>>
    RetryLogic --> SendApiRequest: Спроба < 3
    RetryLogic --> LogError: Спроби вичерпано

    LogError --> [*]
```
