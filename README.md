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
