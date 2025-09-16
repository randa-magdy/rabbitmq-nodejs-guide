# RabbitMQ Documentation

## Table of Contents

1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Exchange Types and Message Flow](#exchange-types-and-message-flow)
   - [Direct Exchange](#direct-exchange)
   - [Fanout Exchange](#fanout-exchange)
   - [Topic Exchange](#topic-exchange)
   - [Headers Exchange](#headers-exchange)
4. [Reliability and Durability](#reliability-and-durability)
   - [Acknowledgements](#acknowledgements-acks)
   - [Message Persistence](#message-persistence)
   - [Publisher Confirms](#publisher-confirms)
5. [Node.js Examples with amqplib](#nodejs-examples-with-amqplib)
   - [Basic Setup](#basic-setup)
   - [Producer Example](#producer-example)
   - [Consumer Example](#consumer-example)
   - [Topic Exchange Example](#topic-exchange-example)
   - [Reliable Publishing](#reliable-publishing)
   - [Quality of Service (QoS)](#quality-of-service-qos)
6. [Advanced Features](#advanced-features)
   - [Dead Letter Exchanges](#dead-letter-exchanges)
   - [Message TTL](#message-ttl-time-to-live)
   - [Consumer Prefetch](#consumer-prefetch)
   - [Queue Limits](#queue-limits-and-overflow-behavior)
   - [Consumer Priorities](#consumer-priorities)
7. [Multi-Tenancy and Security](#multi-tenancy-and-security)
   - [Virtual Hosts](#virtual-hosts-vhost)
   - [User Permissions](#user-permissions)
8. [High Availability](#high-availability)
   - [Quorum Queues](#quorum-queues)
   - [Clustering](#clustering)
   - [Single Active Consumer](#single-active-consumer-sac)
9. [Streaming](#streaming)
   - [RabbitMQ Streams](#rabbitmq-streams)
   - [Super Streams](#super-streams)
10. [Integration and Plugins](#integration-and-plugins)
    - [Federation Plugin](#federation-plugin)
    - [Shovel Plugin](#shovel-plugin)
    - [MQTT Plugin](#mqtt-plugin-iot)
11. [Best Practices](#best-practices)
12. [Summary](#summary)

## Introduction

RabbitMQ is a robust, open-source message broker that enables asynchronous communication between applications. It acts as an intermediary, allowing different parts of a system to communicate by sending and receiving messages without direct coupling.

### Key Features
- **Reliability**: Message persistence and acknowledgements ensure no data loss
- **Flexible Routing**: Multiple exchange types for different messaging patterns
- **Clustering**: High availability and horizontal scaling
- **Multi-Protocol Support**: AMQP, MQTT, STOMP, and more
- **Management UI**: Web-based administration interface

### Why Use RabbitMQ?
- **Decoupling**: Services can operate independently without direct connections
- **Scalability**: Distribute workload across multiple consumers
- **Resilience**: System continues operating even if individual components fail
- **Event-Driven Architecture**: Enable reactive, loosely-coupled systems

## Core Concepts

Think of RabbitMQ as a postal service system:

- **Message Broker (RabbitMQ)**: The postal service that routes messages
- **Producer/Publisher**: The sender of messages
- **Consumer**: The receiver of messages  
- **Queue**: A mailbox that stores messages until they're consumed
- **Exchange**: The sorting facility that routes messages to appropriate queues
- **Binding**: Rules that connect exchanges to queues
- **Routing Key**: Message attribute used for routing decisions (like a zip code)

### Message Flow
The fundamental flow is always: **Producer → Exchange → Queue → Consumer**

The exchange uses its type and bindings to determine how to route messages to queues.

## Exchange Types and Message Flow

### Default Exchange
The default exchange is a direct exchange that has several special properties:

It always exists (is pre-declared)
Its name for AMQP 0-9-1 clients is an empty string ("")
When a queue is declared, RabbitMQ will automatically bind that queue to the default exchange using its (queue) name as the routing key

**Note**: 
- The default exchange is used for its special properties. It is not supposed to be used as "regular" exchange that applications explicitly create bindings for.
- For such cases where a direct exchange and a custom topology are necessary, consider declaring and using a separate direct exchange
  
```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        DirectExchange["AMQP default"]
  end
    Publisher["Publisher"] -- Publish --> DirectExchange
    DirectExchange -- q1 --> Queue1
    Queue1 -- Consume --> Consumer["Consumer"]
    Exchange@{ label: "Exchange : ''" }
    log["key : q1"]

    Exchange@{ shape: rect}
    style Queue1 fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
    style Exchange border:none,stroke-width:2px,stroke-dasharray: 2
    style log stroke-width:2px,stroke-dasharray: 2
```

**Flow:**

1. Publisher sends a message to the default exchange ("") with a routing key set to a specific queue's name (e.g., email-queue).
2. The default exchange (a direct type) examines the message's routing key.
3. It finds the queue that is bound with a routing key that is an exact match for the queue name (e.g., finds queue email-queue bound with key email-queue).
4. The exchange delivers the message only to that single queue.
5. Consumers subscribed to that queue receive the message.

**In short**: Publishing to the default exchange with routing key X delivers the message directly and exclusively to the queue named X.
**Use Case**: Simple, single-purpose apps

### Direct Exchange

Routes messages to queues based on exact routing key matches.

```mermaid
flowchart LR
    Publisher["Publisher"] -- Publish --> DirectExchange["amq.direct"]
    log["key : log"]
    subgraph "RabbitMQ"
    DirectExchange -- log --> Queue1["q1"] & Queue2["q2"] 
    DirectExchange --> Queue3["q3"]
    end
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 -- Consume --> Consumer
    
    style Publisher fill:#c53030,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style Consumer fill:#c53030,color:#fff
    style log stroke-width:2px,stroke-dasharray: 2
```

**Flow:**
1. Publisher sends message to direct exchange with routing key `log`
2. Exchange examines all bound queues
3. Delivers message to queues bound with matching routing key `log`
4. Consumers receive messages from their respective queues

**Use Case**: Point-to-point messaging where specific message types go to specific queues.

### Fanout Exchange

Broadcasts messages to all bound queues, ignoring routing keys (No routing key is assigned).

```mermaid
flowchart LR
    Publisher["Publisher"] -- Publish --> DirectExchange["amq.fanout"]
    subgraph "RabbitMQ"
    DirectExchange --> Queue1["q1"] & Queue2["q2"] 
    Queue3["q3"]
    end 
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 
    
    style Publisher fill:#c53030,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style Consumer fill:#c53030,color:#fff
```

**Flow:**
1. Publisher sends message to fanout exchange
2. Exchange copies message to every bound queue
3. All consumers receive the message

**Use Case**: Pub/Sub scenarios where all subscribers need every message.

### Topic Exchange

Routes messages using pattern matching with wildcards:
- `*` matches exactly one word
- `#` matches zero or more words

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        Queue2["q2"]
        Queue3["q3"]
        DirectExchange["amq.topic"]
  end
  
    Publisher["Publisher"] -- Publish --> DirectExchange
    log["key : log.info"]
    DirectExchange --log.info --> Queue1
    DirectExchange --log.*--> Queue3
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 -- Consume --> Consumer

    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
```

```mermaid

flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        Queue2["q2"]
        Queue3["q3"]
        DirectExchange["amq.topic"]
  end
  
    Publisher["Publisher"] -- Publish --> DirectExchange
    log["key : log.error"]
    DirectExchange --log.error--> Queue2
    DirectExchange --log.*--> Queue3
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 -- Consume --> Consumer

    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
```

**Example Bindings:**
- `q1`: bound to `log.info`
- `q2`: bound to `log.error`  
- `q3`: bound to `log.*`

**Scenarios:**
- Message with route key `log.info` → routed to `q1` and `q3`
- Message with route key `log.error` → routed to `q2` and `q3`

**Use Case**: Flexible routing based on hierarchical topics (e.g., logging levels, geographic regions).

### Headers Exchange

Routes based on message headers rather than routing keys.

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        Queue2["q2"]
        Queue3["q3"]
        DirectExchange["amq.headers"]
  end
  
    Publisher["Publisher"] -- Publish --> DirectExchange
    log["header(s) : info:1"]
    DirectExchange --error:1 <br/> info:1--> Queue3
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 -- Consume --> Consumer

    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
```

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        Queue2["q2"]
        Queue3["q3"]
        DirectExchange["amq.headers"]
  end
  
    Publisher["Publisher"] -- Publish --> DirectExchange
    log["header(s) : error:1"]
    DirectExchange --error:1--> Queue2
    DirectExchange --error:1 <br/> info:1--> Queue3
    Queue1 -- Consume --> Consumer["Consumer"]
    Queue2 -- Consume --> Consumer
    Queue3 -- Consume --> Consumer

    style Queue1 fill:#9f7aea,color:#fff
    style Queue2 fill:#9f7aea,color:#fff
    style Queue3 fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
```

**Configuration:**
- `x-match: all` - must match all specified headers
- `x-match: any` - must match at least one header

**Example:**
- Queue bound with `{error: 1, info: 1, x-match: all}` → Receives messages containing both `error: 1` AND `info: 1` headers.
- Queue bound with `{error: 1, info: 1, x-match: any}` → Receives messages containing either `error: 1` OR `info: 1` header.
  
**Use Case**: Complex routing based on multiple message attributes.

## Reliability and Durability

### Acknowledgements (Acks)

Messages aren't automatically removed from queues when delivered. Consumers must acknowledge successful processing.

- **Positive Ack (`channel.ack(msg)`)**: Confirms successful processing, removes message
- **Negative Ack (`channel.nack(msg)`)**: Indicates failure, can requeue for retry
- **Auto-Ack (`noAck: true`)**: ⚠️ **Risky** - automatically removes messages on delivery

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["q1"]
        DirectExchange["Exchange"]
  end
    Publisher["Publisher"] -- Publish --> DirectExchange
    DirectExchange --> Queue1
    Queue1 -- Consume --> Consumer["Consumer"]
    Consumer --Ack / N-Ack -->Queue1

     style Queue1 fill:#9f7aea,color:#fff
     style DirectExchange fill:#000,color:#fff
     style Publisher fill:#c53030,color:#fff
     style Consumer fill:#c53030,color:#fff
```

### Message Persistence

Understanding durability is key to building a reliable system. It involves two layers: the queue and the message itself. 
 - **Queue Durability:** A durable queue (durable: true) will survive a broker restart. Its definition is saved to disk. A transient queue (durable: false) will be deleted on broker restart. 
 - **Message Delivery Mode:** A message can be persistent (saved to disk) or non-persistent (only stored in memory for performance). 

**Crucial Interaction:** The final persistence of a message depends on both the queue's durability and its own delivery mode. 

| Queue Type | Message Delivery Mode  | Resulting Message State |
|------------|--------------|---------|
| Durable | Persistent | Persistent (on disk) |
| Durable | Non-persistent | Non-persistent (in memory only)  |
| Transient | Persistent | Non-persistent (lost on restart)  |
| Transient | Non-persistent | Non-persistent (lost on restart) |

**Key Rule**: Messages are truly persistent only when marked persistent AND stored in durable queues.

**Non-presistent Message**

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["transient"]
        Exchange["Exchange"]
  end
    Publisher["Publisher"] -- Publish --> Exchange
    Exchange --> Queue1
    Queue1 --> Memory["Memory"]

    style Queue1 fill:#9f7aea,color:#fff
    style Exchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Memory fill:#c53030,color:#fff
```

**Presistent Message**

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue1["durable-q"]
        Exchange["Exchange"]
  end
    Publisher["Publisher"] -- Publish --> Exchange
    Exchange --> Queue1
    Queue1 --> Memory["Memory"]
    Queue1 --> Disk["Disk"]

    style Queue1 fill:#9f7aea,color:#fff
    style Exchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Memory fill:#c53030,color:#fff
    style Disk fill:#c53030,color:#fff
```

### Publisher Confirms

Ensures messages reach the broker successfully:

1. Enable confirmation mode on channel
2. Publish message
3. Broker responds with `basic.ack` (success) or `basic.nack` (failure)

## Node.js Examples with amqplib

### 1. Setup and Connection

First, install the library:

```bash
npm install amqplib
````

### 2. Producer: Sending to a Direct Queue

This example connects, creates a channel, declares a queue, and sends a single message.

```js
const amqp = require('amqplib');

async function produce() {
  try {
    // 1. Connect to the RabbitMQ server
    const connection = await amqp.connect('amqp://localhost');
    const channel = await connection.createChannel();

    // 2. Declare a queue. If it doesn't exist, it will be created.
    // durable: true means the queue will survive a broker restart
    const queueName = 'my_direct_queue';
    await channel.assertQueue(queueName, { durable: true });

    // 3. Define the message
    const message = 'Hello, RabbitMQ from Node.js!';

    // 4. Send the message to the queue
    // The empty string '' as the exchange name denotes the default exchange.
    // The default exchange is a direct exchange that routes messages to the queue named by the routingKey.
    channel.sendToQueue(queueName, Buffer.from(message), { persistent: true });
    console.log(" [x] Sent '%s'", message);

    // 5. Close the connection after a short delay
    setTimeout(() => {
      connection.close();
      process.exit(0);
    }, 500);
  } catch (error) {
    console.error(error);
  }
}

produce();
```

### 3. Consumer: Receiving from a Queue

This example connects, creates a channel, declares the same queue, and sets up a consumer to listen for messages.

```js
const amqp = require('amqplib');

async function consume() {
  try {
    // 1. Connect to the RabbitMQ server
    const connection = await amqp.connect('amqp://localhost');
    const channel = await connection.createChannel();

    // 2. Declare the same queue. This is idempotent.
    const queueName = 'my_direct_queue';
    await channel.assertQueue(queueName, { durable: true });

    console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", queueName);

    // 3. Set up a consumer
    // { noAck: false } means we will manually send acknowledgements.
    channel.consume(queueName, (msg) => {
      if (msg !== null) {
        console.log(" [x] Received '%s'", msg.content.toString());

        // For now, just ack immediately
        channel.ack(msg); // Acknowledge the message processing is done
      }
    }, { noAck: false }); // Manual acknowledgement mode

  } catch (error) {
    console.error(error);
  }
}

consume();
```

### 4. Using a Topic Exchange

This example demonstrates the powerful **topic exchange pattern**.

**Producer:**

```js
// generateLogs.js
const amqp = require('amqplib');

async function generateLogs() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // Declare the topic exchange
  const exchangeName = 'amq.topic'; // Or use 'logs_topic'
  await channel.assertExchange(exchangeName, 'topic', { durable: true });

  const logLevels = ['info', 'warning', 'error'];
  let count = 0;

  setInterval(() => {
    count++;
    // Choose a random log level
    const severity = logLevels[Math.floor(Math.random() * 3)];
    const message = `Log ${severity} event #${count}`;
    const routingKey = `log.${severity}`;

    // Publish to the exchange with a routing key
    channel.publish(exchangeName, routingKey, Buffer.from(message));
    console.log(" [x] Sent '%s' with key '%s'", message, routingKey);
  }, 2000);
}

generateLogs();
```

**Consumer:**

```js
// logConsumer.js
const amqp = require('amqplib');

async function createConsumer(severityPattern, consumerTag) {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchangeName = 'amq.topic';

  // Assert the exchange
  await channel.assertExchange(exchangeName, 'topic', { durable: true });

  // Assert an exclusive, auto-delete queue. This is common for temporary consumers.
  const q = await channel.assertQueue('', { exclusive: true });

  // Create bindings based on the pattern
  console.log(`Consumer ${consumerTag} waiting for logs with pattern '${severityPattern}'`);
  await channel.bindQueue(q.queue, exchangeName, severityPattern);

  channel.consume(q.queue, (msg) => {
    if (msg) {
      console.log(` [${consumerTag}] ${msg.fields.routingKey}: '${msg.content.toString()}'`);
      channel.ack(msg);
    }
  }, { noAck: false });
}

// Create three different consumers for different patterns
createConsumer('log.info', 'INFO_READER');
createConsumer('log.error', 'ERROR_READER');
createConsumer('log.*', 'ALL_LOGS_READER'); // Will get both info and error logs
```

### Reliable Publishing

How can a publisher be sure a message actually reached RabbitMQ? The basic publish method is a "fire-and-forget" operation. For reliability, you need Publisher Confirms. 

**message successfully handled**

```mermaid
flowchart LR
    Publisher--Publish(1)-->RabbitMQ
    RabbitMQ --Confirm(2) -->  Publisher

    style Publisher fill:#c53030,color:#fff
    style RabbitMQ fill:#f27c1f,color:#fff
```

**message failure**

```mermaid
flowchart LR
    Publisher--Publish(1)-->RabbitMQ--Return(2)-->Publisher
    RabbitMQ --Error(3 -->  Publisher
    
    style Publisher fill:#c53030,color:#fff
    style RabbitMQ fill:#f27c1f,color:#fff
```

**How it works:**
- The publisher puts the channel into confirm mode. 
- It publishes a message. 
- RabbitMQ responds with a basic.ack (confirm) if the message was successfully handled by the broker (e.g., routed to a queue). 
- RabbitMQ responds with a basic.nack or a basic.return (if mandatory is set) if the message could not be processed or routed, indicating a failure. 

This mechanism ensures messages are not lost on their way to the broker. 

```js
const amqp = require('amqplib');

async function reliableProduce() {
  try {
    const connection = await amqp.connect('amqp://localhost');
    const channel = await connection.createChannel();

    // Enable publisher confirms
    await channel.confirmSelect();

    // Listen for confirmations
    channel.on('ack', () => console.log('Message confirmed by broker'));
    channel.on('nack', () => console.error('Message rejected by broker'));
    
    const queueName = 'durable_important_queue';
    await channel.assertQueue(queueName, { durable: true });

    const message = 'Very important persistent message';
    const options = { persistent: true };

    const isSent = channel.sendToQueue(queueName, Buffer.from(message), options);
    if (isSent) {
      console.log(" [x] Sent '%s' (waiting for confirm...)", message);
    }

    // Optional: wait for all confirms before closing
    await channel.waitForConfirms();
    
  } catch (error) {
    console.error(error);
  }
}

reliableProduce();
```

## Advanced Features

### Dead Letter Exchanges

Handle messages that cannot be processed by routing them to a Dead Letter Exchange (DLX).

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        DeadQueue["dead-queue"]
        DirectExchange["Exchange"]
  end
    Publisher["Publisher"] -- Publish --> DirectExchange
    DirectExchange --> DeadQueue
    DeadQueue -- Consume --> Consumer["Consumer"]
    Consumer -- Reject without requeue --> DeadExchange["Dead Exchange"]
    DeadExchange --> Queue1["q1"]

    style DeadQueue fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Consumer fill:#c53030,color:#fff
    style DeadExchange fill:#000,color:#fff
    style Queue1 fill:#9f7aea,color:#fff
```

**How it works:**
- Configure a queue (main-queue) to forward failed messages to a special exchange (the DLX) by setting the x-dead-letter-exchange argument. 
- You can optionally set x-dead-letter-routing-key. 
- Create another queue (dead-letter-queue) bound to this DLX. 
- When a message in main-queue is rejected (channel.nack) or exceeds its retry limit, it is automatically rerouted to the DLX and then to the dead-letter-queue. 
- You can then have a separate process to analyze the messages in the DLQ.
  
**When to use it:** For implementing retry logic, debugging failing messages, and handling poison pills (messages that consistently cause consumers to crash).

**Setup:**

```js
const amqp = require('amqplib');

async function setupDLQ() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // Declare Dead Letter Exchange and Queue
  const dlxName = 'my_dlx';
  const dlqName = 'dead_letter_queue';
  await channel.assertExchange(dlxName, 'direct', { durable: true });
  await channel.assertQueue(dlqName, { durable: true });
  await channel.bindQueue(dlqName, dlxName, '');

  // Main queue with DLX configuration
  const mainQueueName = 'work_queue_with_retries';
  await channel.assertQueue(mainQueueName, {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': dlxName,
      'x-message-ttl': 60000, // Optional: expire messages after 1 minute
      'x-dead-letter-routing-key': 'failed.message' // Optional new routing key
    }
  });

  console.log(" [x] DLQ setup complete.");
  await connection.close();
}

setupDLQ();
```

**Consumer with Retry Logic:**

```js
async function consumeWithRetries() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  await channel.prefetch(1);

  const mainQueueName = 'work_queue_with_retries';

  channel.consume(mainQueueName, (msg) => {
    const messageContent = msg.content.toString();
    console.log(" [x] Processing: '%s'", messageContent);

    // Simulate random failure 30% of the time
    if (Math.random() < 0.3) {
      console.log(" [x] Processing FAILED. Sending to DLQ.");
      // Negative acknowledge, don't requeue - goes to DLX
      channel.nack(msg, false, false);
    } else {
      console.log(" [x] Processing SUCCESSFUL.");
      channel.ack(msg);
    }
  }, { noAck: false });
}

consumeWithRetries();
```

### Message TTL (Time To Live)

Messages can expire after a specified time period.

```mermaid
flowchart LR
 subgraph RabbitMQ["RabbitMQ"]
        Queue["time-msg"]
        DirectExchange["amq.direct"]
  end
    Publisher["Publisher"] -- Publish --> DirectExchange
    DirectExchange --> Queue
    Queue -- Expired --> Exchange2["amq.fanout"]
    Exchange2 --> Queue1["expired-messages"]
    Time["Time"]

    style Queue fill:#9f7aea,color:#fff
    style DirectExchange fill:#000,color:#fff
    style Publisher fill:#c53030,color:#fff
    style Exchange2 fill:#000,color:#fff
    style Queue1 fill:#9f7aea,color:#fff
    style n1 stroke:#FF6D00
```

**How it works:** You can set a TTL either on a queue (applies to all messages in the queue) or on a per-message basis. 

**What happens:** When a message expires, it is automatically removed from the queue and, if configured, dead-lettered to a Dead Letter Exchange (DLX). This is useful for time-sensitive operations like "hold a booking for 15 minutes." 

```js
// Set TTL on queue (applies to all messages)
await channel.assertQueue('expiring_queue', {
  durable: true,
  arguments: {
    'x-message-ttl': 60000 // 60 seconds
  }
});

// Set TTL on individual message
channel.sendToQueue('my_queue', Buffer.from('Expiring message'), {
  persistent: true,
  expiration: '30000' // 30 seconds (string format)
});
```

### Consumer Prefetch (Quality of Service - QoS)
Prefetch controls how many messages are sent to a consumer before they are acknowledged. This is critical for balancing load. 

```mermaid
flowchart LR
    RabbitMQ["RabbitMQ"] -- "Unacked &lt;= Prefetch Count" --> Consumer["Consumer"]
    Consumer --> RestApis["RestApis"] & Database["Database"]

    style RabbitMQ fill:#f27c1f,color:#fff
    style Consumer fill:#c53030,color:#fff
    style RestApis fill:#FFD600
    style Database fill:#00C853
```

**The Problem:** Without a limit, RabbitMQ will send as many messages as it can to a consumer, potentially overwhelming it while other consumers sit idle. 

**The Solution:** Set a prefetch count. This defines the maximum number of **unacknowledged messages** a consumer can have at any time. 
  - **Example:** If prefetchCount = 1, RabbitMQ will send the next message to the consumer only after the previous one has been acked. This ensures fair distribution and prevents any single consumer from being flooded. 

```js
const amqp = require('amqplib');

async function consumeWithQoS() {
  try {
    const connection = await amqp.connect('amqp://localhost');
    const channel = await connection.createChannel();

    // Set prefetch to 1 - only one unacknowledged message at a time
    await channel.prefetch(1);

    const queueName = 'work_queue';
    await channel.assertQueue(queueName, { durable: true });

    console.log(" [*] Waiting for messages. Prefetch is 1.");

    channel.consume(queueName, async (msg) => {
      if (msg !== null) {
        console.log(" [x] Received '%s'", msg.content.toString());

        // Simulate variable processing time
        const workDuration = Math.random() * 5000;
        await new Promise(resolve => setTimeout(resolve, workDuration));

        // Acknowledge after work is done
        channel.ack(msg);
        console.log(" [x] Done. Message acknowledged.");
      }
    }, { noAck: false });

  } catch (error) {
    console.error(error);
  }
}

consumeWithQoS();
```

### Queue Capacity & Overflow Behavior

Queues can have a maximum length to prevent them from growing indefinitely and consuming all disk/memory. 

**Max Length:** The maximum number of messages a queue can hold. 

**Overflow Behavior:** What happens when a queue is full? 
   - **drop-head (default):** Delete the oldest message at the front of the queue to make space for the new one. 
   - **reject-publish:** The broker will **reject** the new message (and trigger a nack to the publisher if using confirms). This is safer for ensuring no messages are silently dropped. 

```js
await channel.assertQueue('limited_queue', {
  durable: true,
  arguments: {
    'x-max-length': 1000, // Maximum 1000 messages
    'x-overflow': 'reject-publish' // Reject new messages when full
    // Alternative: 'x-overflow': 'drop-head' (remove oldest)
  }
});
```

### Consumer Priorities

You can assign priorities to consumers. If multiple consumers are idle, the broker will preferentially send new messages to the consumer with the highest priority. 

**Example:** Consumer 1 (prio=10), Consumer 2 (prio=5), Consumer 3 (prio=0). 
**Result:** If all are idle, messages will be delivered first to Consumer 1, then 2, then 3. This is useful for giving more power to stronger machines or more important processing services. 

 
```js
channel.consume(queueName, handleMessage, {
  noAck: false,
  priority: 10 // Higher priority consumer
});

channel.consume(queueName, handleMessage, {
  noAck: false,
  priority: 5 // Lower priority consumer
});
```

## Multi-Tenancy and Security

### Virtual Hosts (vhost)

Virtual hosts provide logical separation within a single RabbitMQ instance:

```js
// Connect to specific virtual host
const connection = await amqp.connect('amqp://username:password@localhost/my-app-prod');
```

**Benefits:**
- Isolate resources between applications
- Separate development, staging, and production environments
- Control resource usage per tenant

### User Permissions

RabbitMQ permissions are granted per user per virtual host:

- **Configure**: Create/destroy resources (queues, exchanges)
- **Write**: Publish messages
- **Read**: Consume messages

**Topic Permissions**: Fine-grained control over routing keys for publish/consume operations.

## High Availability

### Quorum Queues

Modern, replicated queue type built on Raft consensus algorithm. **Recommended for clustered setups.**

**Configuration:**
Quorum queues are declared with a special argument and have their own settings. 
 - **x-queue-type: quorum:** The main argument to create a quorum queue.
 - **x-quorum-initial-group-size:** The number of replicas for the queue (e.g., 3 for 2 tolerable node failures). 
 - **x-dead-letter-strategy:** Can be set to at-least-once (more reliable DLQ routing) instead of the default at-most-once. 

```js
await channel.assertQueue('ha-payment-queue', {
  durable: true,
  arguments: {
    'x-queue-type': 'quorum', // Enable quorum queue
    'x-quorum-initial-group-size': 3, // Replicate across 3 nodes
    'x-delivery-limit': 3 // Max delivery attempts before dead-lettering
  }
});
```

**Key Features:** 
- **Replicated by Design:** Messages are synchronously replicated across a quorum (majority) of nodes in the cluster. This ensures data is not lost if a node fails. 
- **Durable and Persistent:** They are always durable, and messages are always persistent. The durable and persistent flags are effectively always true. 
- **Poison Message Handling:** If a message is negatively acknowledged (nack) and requeued multiple times (default: 3 times), Quorum Queues will automatically dead-letter it or drop it. This prevents a single broken message from blocking a queue indefinitely. 
- **Lazy by Default:** They always write messages to disk first, preventing memory overload. 

**Classic Queue vs Quorum Queue**

| Feature                  | Classic   | Quorum   |
|---------------------------|-----------|----------|
| Non-durable queues        | Yes       | No       |
| Exclusivity               | Yes       | No       |
| Per message persistence   | Yes       | Always   |
| Membership changes        | Automatic | Manual   |
| Message TTL               | Yes       | Yes      |
| Queue TTL                 | Yes       | Yes      |
| Queue length limits       | Yes       | Yes      |
| Lazy behavior             | Yes       | Always   |
| Message priority          | Yes       | No       |
| Consumer priority         | Yes       | Yes      |
| Dead letter exchanges     | Yes       | Yes      |
| Adheres to policies       | Yes       | Yes      |
| Poison message handling   | No        | Yes      |
| Global QoS Prefetch       | Yes       | No       |
 

### Clustering

Multiple RabbitMQ nodes working together:

```bash
# Start two nodes and create cluster
set RABBITMQ_NODENAME=rabbit@node01
rabbitmq-server -detached

set RABBITMQ_NODENAME=rabbit@node02  
rabbitmq-server -detached

# Join node02 to node01's cluster
rabbitmqctl -n rabbit@node02 stop_app
rabbitmqctl -n rabbit@node02 join_cluster rabbit@node01
rabbitmqctl -n rabbit@node02 start_app
```

### Single Active Consumer (SAC)

Provides automatic consumer failover:

```js
// Enable SAC on queue
await channel.assertQueue('my-sac-queue', {
  arguments: { 'x-single-active-consumer': true }
});
```

Only one consumer in the group is active; others are standby until the active consumer fails.

## Streaming

### RabbitMQ Streams

Append-only log designed for high throughput and message replay:

```bash
npm install rabbitmq-streams-client
```

**Stream Producer:**

```js
const { Client } = require('rabbitmq-streams-client');

async function produceToStream() {
  const client = new Client('rabbitmq-stream://localhost:5552');
  await client.connect();

  const streamName = 'application-log-stream';
  
  // Create stream with retention policies
  await client.createStream(streamName, {
    'max-age': '2D', // Keep for 2 days
    'max-length-bytes': 5000000000 // ~5 GB max size
  });

  const producer = await client.declarePublisher({ stream: streamName });

  for (let i = 0; i < 1000; i++) {
    const message = {
      level: i % 2 === 0 ? 'INFO' : 'ERROR',
      message: `Log event #${i}`,
      timestamp: new Date().toISOString()
    };

    await producer.send(Buffer.from(JSON.stringify(message)), {
      applicationProperties: { 'level': message.level } // For filtering
    });
  }

  await client.close();
}

produceToStream();
```

**Stream Consumer with Filtering:**

```js
const { Client, Offset } = require('rabbitmq-streams-client');

async function consumeErrorsOnly() {
  const client = new Client('rabbitmq-stream://localhost:5552');
  await client.connect();

  const consumer = await client.declareConsumer({
    stream: 'application-log-stream',
    name: 'error-log-processor',
    offset: Offset.first(), // Start from beginning for replay
    filter: {
      values: { 'level': 'ERROR' }, // Only ERROR messages
      matchUnfiltered: false
    },
    callback: async (message, context) => {
      const logEntry = JSON.parse(message.content.toString());
      console.error(` [!] CRITICAL ERROR: ${logEntry.message}`);
      context.storeOffset(); // Track progress
    }
  });

  console.log(" [*] Error consumer started, filtering for 'ERROR' level logs.");
}

consumeErrorsOnly();
```

### Super Streams

Partitioned streams for horizontal scaling:

```bash
# Create super stream with 3 partitions
rabbitmq-streams add_super_stream orders --partitions 3
```

**Producer with Routing:**

```js
async function produceToSuperStream() {
  const client = new Client('rabbitmq-stream://localhost:5552');
  await client.connect();

  const producer = await client.declarePublisher({ 
    stream: 'orders', 
    superStream: true 
  });

  for (let i = 0; i < 10; i++) {
    const order = {
      orderId: i,
      customerId: `customer_${i % 3}`, // Routing key
      amount: 100 * i
    };

    // Messages with same customerId go to same partition
    await producer.send(Buffer.from(JSON.stringify(order)), {
      routingKey: order.customerId
    });
  }

  await client.close();
}
```

## Integration and Plugins

### Federation Plugin

Links exchanges and queues across different brokers or networks:

- **Upstream/Downstream model**: Downstream pulls from upstream
- **Tolerates network failures**: Buffers and reconnects automatically
- **Use case**: Branch office to data center connectivity

### Shovel Plugin

Moves messages from one broker to another:

- **Point-to-point relay**: More like a persistent client
- **Migration tool**: Move data between clusters
- **Use case**: Cloud migration, edge-to-cloud data flow

### MQTT Plugin (IoT)

Enables RabbitMQ to act as MQTT broker:

```bash
rabbitmq-plugins enable rabbitmq_mqtt
```

**Features:**
- QoS levels 0, 1 (QoS 2 downgraded to 1)
- Last Will and Testament (LWT)
- Retained messages
- Maps MQTT topics to AMQP exchanges

## Best Practices

1. **Always Use Manual Acknowledgements** (`noAck: false`) for reliability
2. **Set Prefetch Limits** (`channel.prefetch(N)`) for fair load distribution
3. **Use Publisher Confirms** for critical messages
4. **Implement Dead Letter Queues** for error handling
5. **Choose Quorum Queues** for clustered, high-availability scenarios  
6. **Use Virtual Hosts** for environment separation
7. **Apply Principle of Least Privilege** for user permissions
8. **Set Queue Limits** to prevent unbounded growth
9. **Use Streams** for event sourcing and audit logs
10. **Plan Partitioning Strategy** carefully for Super Streams

## Summary

RabbitMQ provides a comprehensive messaging platform that scales from simple point-to-point messaging to complex, distributed streaming architectures:

### Exchange Types Comparison

| Exchange Type | Routing Method | Use Case |
|---------------|----------------|----------|
| **Direct** | Exact key match | Point-to-point, RPC |
| **Fanout** | Broadcast to all | Pub/Sub, notifications |
| **Topic** | Pattern matching | Hierarchical routing |
| **Headers** | Header attributes | Complex multi-criteria routing |

### Queue Types Comparison

| Feature | Classic Queue | Quorum Queue | Stream |
|---------|---------------|--------------|---------|
| **Data Model** | FIFO Queue | FIFO Queue | Append Log |
| **Consumption** | Destructive (Ack) | Destructive (Ack) | Non-destructive (Read) |
| **Replay** | No | No | Yes |
| **Clustering** | Via policies | Built-in Raft | Built-in |
| **Use Case** | Traditional queuing | HA queuing | Event sourcing, audit |

### Architecture Patterns

- **Microservices**: Direct exchanges for RPC, topic exchanges for events
- **Event-Driven Systems**: Streams for event sourcing, queues for command handling  
- **IoT Platforms**: MQTT plugin for device connectivity, streams for telemetry
- **Multi-Tenant SaaS**: Virtual hosts for isolation, topic permissions for security
- **Global Systems**: Federation for geographic distribution, clustering for local HA

RabbitMQ's rich feature set enables building robust, scalable, and maintainable distributed systems that handle everything from real-time user requests to large-scale data analytics pipelines.
