# 5 - Application State & Network Connectivity

## 5.1 - Application State Design

Application state should be designed around how data is accessed, searched, updated, stored, and removed from memory.

### 5.1.1 - State Classification

![Application state organized by data classes and data properties](assets/images/2026-07-07-14-13-11.png)

State can be classified by its role and by its runtime characteristics.

| Category          | Examples                                     |
| ----------------- | -------------------------------------------- |
| App configuration | Theme, locale, font size, accessibility      |
| UI state          | Selected controls, open panels, entered text |
| Server data       | Users, posts, messages, conversations        |

| Property             | Main question                                    |
| -------------------- | ------------------------------------------------ |
| Access scope         | Is the data local or global?                     |
| Read/write frequency | How often is it accessed or changed?             |
| Size                 | Can it remain in memory safely?                  |
| Lifetime             | Is it temporary, session-based, or persistent?   |
| Search needs         | Is lookup by ID enough, or is indexing required? |

These properties determine both the state structure and the storage mechanism.

### 5.1.2 - Main Design Goals

A good state structure minimizes three costs:

| Goal                    | Preferred approach                      |
| ----------------------- | --------------------------------------- |
| Fast direct access      | Store entities by ID                    |
| Fast search             | Add indexes when needed                 |
| Controlled memory usage | Keep only active data in runtime memory |

Direct access by ID is usually faster than scanning an array.

```ts
const message = state.messagesById[messageId];
```

This is preferable to:

```ts
const message = messages.find((message) => message.id === messageId);
```

### 5.1.3 - Data Normalization

Normalization stores each entity once and uses IDs to represent relationships.

#### 5.1.3.1 - First Normal Form

![Conversion from nested non-normalized data to first normal form](assets/images/2026-07-07-14-14-43.png)

For UI state, the practical rules are:

- each entity has a stable primary key
- nested independent entities are replaced by atomic fields or IDs
- deeply nested reusable data is avoided

Example:

```ts
type User = {
  id: string;
  name: string;
  jobId: string;
  countryCode: string;
};
```

#### 5.1.3.2 - Second Normal Form

![Separation of user and job entities when moving toward second normal form](assets/images/2026-07-07-14-15-09.png)

Data that belongs to another entity is moved to its own collection.

```ts
type User = {
  id: string;
  name: string;
  jobId: string;
};

type Job = {
  id: string;
  title: string;
  department: string;
};
```

This avoids duplicating job data across multiple users.

#### 5.1.3.3 - Third Normal Form

![Separation of job and department data when moving toward third normal form](assets/images/2026-07-07-14-15-27.png)

Fields that belong to a separate entity are extracted again.

```ts
type Job = {
  id: string;
  title: string;
  departmentId: string;
};

type Department = {
  id: string;
  name: string;
};
```

In UI applications, this level is useful only when the extra separation improves reuse, consistency, or access.

#### 5.1.3.4 - Messenger Example

![Normalization process for contacts, messages, and conversations](assets/images/2026-07-07-14-15-44.png)

The normalization flow is:

1. Remove nested conversations and messages.
2. Replace nested relationships with IDs.
3. Define a primary key for each entity.
4. Store entities in separate keyed collections.

```ts
type AppState = {
  contacts: Record<string, Contact>;
  conversations: Record<string, Conversation>;
  messages: Record<string, Message>;
};
```

Result:

```ts
const message = state.messages[messageId];
```

The entity can now be accessed directly without scanning nested arrays.

### 5.1.4 - Relationship and Search Indexes

Normalization improves lookup by ID, but relationships and text search may still require extra indexes.

For conversation relationships:

```ts
type AppState = {
  messagesById: Record<string, Message>;
  messageIdsByConversationId: Record<string, string[]>;
};
```

This separates two concerns:

| Structure                  | Purpose                             |
| -------------------------- | ----------------------------------- |
| messagesById               | Find one message by ID              |
| messageIdsByConversationId | Find all messages in a conversation |

### 5.1.5 - Inverted Indexes

![Inverted index using exact words and prefix-based keys](assets/images/2026-07-07-14-17-34.png)

An inverted index maps searchable content to entity IDs.

```txt
word or prefix → message IDs
```

Exact-word index:

```ts
const messageIndex = {
  jane: ["1"],
  hey: ["2"],
};
```

Prefix index:

```ts
const prefixIndex = {
  j: ["1"],
  ja: ["1"],
  jan: ["1"],
  jane: ["1"],
};
```

| Benefit            | Cost                   |
| ------------------ | ---------------------- |
| Faster search      | More storage           |
| Prefix matching    | More indexing work     |
| No full array scan | More update complexity |

Large indexes should be built incrementally, in a Web Worker, or during idle time.

### 5.1.6 - Runtime Memory and Persistence

Runtime state should contain the active working set, not every record the application has ever loaded.

```txt
active data → runtime state
inactive data → persistent storage
```

For a messaging app:

- active conversation stays in memory
- inactive conversations move out of runtime state
- persisted data remains in IndexedDB
- selecting a conversation loads it back into memory

Data can be garbage-collected only after all active references are removed.

### 5.1.7 - Browser Storage

![Comparison of IndexedDB, localStorage, and sessionStorage](assets/images/2026-07-07-14-18-33.png)

| Property             | IndexedDB         | localStorage | sessionStorage      |
| -------------------- | ----------------- | ------------ | ------------------- |
| API                  | Asynchronous      | Synchronous  | Synchronous         |
| Data format          | Structured values | Strings      | Strings             |
| Persistence          | Persistent        | Persistent   | Current tab session |
| Indexes              | Yes               | No           | No                  |
| Suitable size        | Medium to large   | Small        | Small               |
| Main-thread blocking | Low at API level  | Yes          | Yes                 |

Important notes:

- IndexedDB is not literally unlimited; browser quota rules apply.
- localStorage and sessionStorage can block the main thread.
- large IndexedDB results can still use significant CPU and memory after loading.

### 5.1.8 - Storage Selection

| Use case                    | Recommended option |
| --------------------------- | ------------------ |
| Active UI data              | Runtime state      |
| Theme and preferences       | localStorage       |
| Temporary per-tab data      | sessionStorage     |
| Large structured data       | IndexedDB          |
| Offline data                | IndexedDB          |
| Indexed search              | IndexedDB          |
| Frequent large reads/writes | IndexedDB          |

A practical design flow is:

```txt
classify data
→ evaluate size, frequency, and lifetime
→ normalize reusable entities
→ add relationship indexes
→ add search indexes only when needed
→ keep active data in memory
→ persist inactive or large data
```

## 5.2 - Network Connectivity

Network connectivity design depends on reliability, latency, connection lifetime, bandwidth, and device constraints.

### 5.2.1 - Latency and Reconnection

![Connection loss and reconnection while moving between mobile network towers](assets/images/2026-07-13-11-38-34.png)

Mobile clients may lose connectivity when moving between network towers or switching networks.

A reconnection may require:

1. detecting the lost connection
2. selecting a new network route
3. establishing a new transport connection
4. restoring application state
5. retrying pending operations

| Responsibility | Typical behavior                                       |
| -------------- | ------------------------------------------------------ |
| Client         | Detects failures, retries, and restores subscriptions  |
| Server         | Keeps enough state to resume the session when required |
| Transport      | Establishes a new connection before data can continue  |

A new TCP connection adds latency because the client and server must complete a handshake before exchanging application data.

### 5.2.2 - UDP and TCP

![Comparison between UDP communication and the TCP three-way handshake](<assets/images/2026-07-13-11-35-52(1).png>)

UDP and TCP provide different transport guarantees.

| Property           | UDP                         | TCP                         |
| ------------------ | --------------------------- | --------------------------- |
| Connection setup   | No handshake                | Three-way handshake         |
| Delivery guarantee | No                          | Yes                         |
| Ordering guarantee | No                          | Yes                         |
| Retransmission     | Application responsibility  | Built into the protocol     |
| Overhead           | Lower                       | Higher                      |
| Typical use        | Real-time media, games, DNS | HTTP/1.1, HTTP/2, WebSocket |

TCP begins with a three-way handshake:

```txt
client → SYN → server
client ← SYN-ACK ← server
client → ACK → server
```

After the connection is established, application data can be exchanged.

UDP avoids this setup cost, but the application must tolerate or handle missing and out-of-order packets.

### 5.2.3 - Protocol Stack

![Relationship between application protocols and TCP or UDP](assets/images/2026-07-13-11-37-04.png)

Web protocols are built on top of transport protocols.

| Protocol           | Transport     | Main use                               |
| ------------------ | ------------- | -------------------------------------- |
| HTTP/1.1           | TCP           | Request-response communication         |
| HTTP/2             | TCP           | Multiplexed HTTP requests              |
| WebSocket          | Usually TCP   | Bidirectional persistent communication |
| Server-Sent Events | HTTP over TCP | Server-to-client event stream          |
| HTTP/3             | QUIC over UDP | Multiplexed HTTP with faster recovery  |
| WebRTC             | Usually UDP   | Real-time audio, video, and peer data  |

Important notes:

- HTTP/3 runs over QUIC, not directly over raw UDP.
- WebSocket starts with an HTTP handshake and then keeps a persistent bidirectional connection.
- Server-Sent Events works over HTTP and is not limited to HTTP/2.
- WebRTC prefers UDP but may fall back to other transports when necessary.

### 5.2.4 - Polling

![Client polling the server for new orders at a fixed interval](assets/images/2026-07-13-11-37-22.png)

Polling repeatedly asks the server whether new data is available.

```ts
const intervalId = window.setInterval(async () => {
  const response = await fetch("/api/orders");
  const orders = await response.json();

  updateOrders(orders);
}, 20_000);
```

Polling is simple, but `data is only discovered on the next request`.

```txt
poll interval = 20 seconds
worst-case update delay ≈ 20 seconds
average update delay ≈ 10 seconds
```

| Advantage                        | Limitation                           |
| -------------------------------- | ------------------------------------ |
| Simple implementation            | Delayed updates                      |
| Works with normal HTTP endpoints | Repeated requests with no new data   |
| Easy server architecture         | Extra headers and network activity   |
| Easy to debug                    | More battery usage on mobile devices |

HTTP connections may be reused through keep-alive, so polling does not always create a new TCP connection for every request. Connection loss or timeout can still force a new handshake.

### 5.2.5 - Mobile Network and Energy Cost

![Connection setup cost and mobile network receive-only and duplex modes](assets/images/2026-07-13-11-37-44.png)

Network activity affects mobile devices more than desktop devices.

Repeated polling can cause:

- periodic CPU work
- radio wake-ups
- repeated request headers
- extra bandwidth usage
- increased battery consumption
- reconnection work after network changes

Mobile radios use more energy while actively sending and receiving data than while waiting in a low-power state.

| Mode                                | Behavior                           | Energy use |
| ----------------------------------- | ---------------------------------- | ---------- |
| Low-power or receive-oriented state | Mostly waits for incoming activity | Lower      |
| Active bidirectional state          | Sends and receives data            | Higher     |

The exact battery cost depends on the device, network, signal quality, request frequency, and operating system. A fixed battery-life estimate should not be treated as universal.

### 5.2.6 - When Polling Is Appropriate

Polling is suitable when:

- updates are infrequent
- a small delay is acceptable
- implementation simplicity is more important than real-time delivery
- the application mainly targets stable desktop connections

Polling is less suitable when:

- the client requires immediate updates
- requests frequently return no new data
- users are on mobile networks
- battery and bandwidth are important
- the connection frequently changes

| Requirement                                   | Better starting point |
| --------------------------------------------- | --------------------- |
| Occasional updates                            | Polling               |
| Server-to-client updates                      | Server-Sent Events    |
| Bidirectional real-time messages              | WebSocket             |
| Real-time media or peer communication         | WebRTC                |
| Modern HTTP with improved connection recovery | HTTP/3                |
