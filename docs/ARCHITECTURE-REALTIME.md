# Teable Real-time Collaboration Architecture

> **Detailed documentation of WebSocket, ShareDB, and event systems.**

## Table of Contents

1. [Overview](#1-overview)
2. [WebSocket Implementation](#2-websocket-implementation)
3. [ShareDB Integration](#3-sharedb-integration)
4. [Event Broadcasting](#4-event-broadcasting)
5. [Event Emitter System](#5-event-emitter-system)
6. [Scalability Considerations](#6-scalability-considerations)

---

## 1. Overview

Teable implements **real-time collaboration** using:

- **ShareDB**: Operational Transformation (OT) for conflict-free concurrent editing
- **Native WebSocket**: Low-latency bidirectional communication
- **Redis PubSub**: Multi-instance synchronization
- **Event System**: Domain events for side effects

### Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client A  │     │   Client B  │     │   Client C  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └─────────┬─────────┴─────────┬─────────┘
                 │                   │
                 ▼                   ▼
         ┌───────────────────────────────────┐
         │        WebSocket Gateway           │
         │   (WsGateway / DevWsGateway)      │
         └────────────────┬──────────────────┘
                          │
                          ▼
         ┌───────────────────────────────────┐
         │         ShareDB Service           │
         │  ┌─────────────────────────────┐ │
         │  │    Custom DB Adapter        │ │
         │  │    Auth Middleware          │ │
         │  │    Presence Support         │ │
         │  └─────────────────────────────┘ │
         └────────────────┬──────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐
    │    Database     │     │   Redis PubSub  │
    │   (Postgres)    │     │  (Multi-node)   │
    └─────────────────┘     └─────────────────┘
```

---

## 2. WebSocket Implementation

### 2.1 Gateway Configuration

**Production Gateway** (`apps/nestjs-backend/src/ws/ws.gateway.ts`):

```typescript
@WebSocketGateway({
  path: '/socket',
  perMessageDeflate: true  // Compression enabled
})
export class WsGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  afterInit(server: Server) {
    server.on('connection', async (webSocket, request) => {
      const stream = new WebSocketJSONStream(webSocket);
      this.shareDb.listen(stream, request);
    });
  }
}
```

**Development Gateway** (`apps/nestjs-backend/src/ws/ws.gateway.dev.ts`):
- Runs on separate port (`SOCKET_PORT`)
- Standalone WebSocket server for hot-reload compatibility

### 2.2 Connection Flow

```
1. Client connects to ws://host/socket
2. WsGateway.afterInit() intercepts connection
3. WebSocketJSONStream wraps the WebSocket
4. ShareDB.listen() handles the stream
5. Auth middleware validates credentials
6. Connection established
```

### 2.3 WebSocket Adapter

Configured in bootstrap:

```typescript
// apps/nestjs-backend/src/bootstrap.ts
app.useWebSocketAdapter(new WsAdapter(app));
```

---

## 3. ShareDB Integration

### 3.1 ShareDB Service

**Location**: `apps/nestjs-backend/src/share-db/share-db.service.ts`

```typescript
export class ShareDbService extends ShareDBClass {
  constructor(shareDbAdapter: ShareDbAdapter, cacheConfig: ICacheConfig) {
    super({
      presence: true,                    // Real-time presence tracking
      doNotForwardSendPresenceErrorsToClient: true,
      db: shareDbAdapter,                // Custom database adapter
      maxSubmitRetries: 3                // Retry failed operations
    });
  }
}
```

### 3.2 Document Types (Collections)

ShareDB organizes data into collections by document type:

| Collection Pattern | Purpose | Example |
|-------------------|---------|---------|
| `t_{tableId}` | Table metadata | `t_tbl_abc123` |
| `f_{tableId}` | Field definitions | `f_tbl_abc123` |
| `v_{tableId}` | View configurations | `v_tbl_abc123` |
| `r_{tableId}` | Record documents | `r_tbl_abc123` |

### 3.3 Custom Database Adapter

**Location**: `apps/nestjs-backend/src/share-db/share-db.adapter.ts`

```typescript
export class ShareDbAdapter extends ShareDb.DB {
  // Fetch document snapshots
  async getSnapshotBulk(collection, ids, projection) {
    const docType = this.getDocType(collection);
    return this.getReadonlyService(docType).getSnapshotBulk(ids, projection);
  }

  // Query documents
  async query(collection, query, projection) {
    const docType = this.getDocType(collection);
    return this.getReadonlyService(docType).query(query, projection);
  }

  // Fetch operation history
  async getOps(collection, id, from, to) {
    // Returns ops from version `from` to `to`
  }
}
```

### 3.4 Authentication Middleware

**Location**: `apps/nestjs-backend/src/share-db/auth.middleware.ts`

```typescript
export const authMiddleware = (shareDB: ShareDBClass) => {
  // Connection hook: Extract auth from HTTP request
  shareDB.use('connect', async (context, callback) => {
    const cookie = context.req.headers.cookie;
    const shareId = new URL(context.req.url).searchParams.get('shareId');

    context.agent.custom.cookie = cookie;
    context.agent.custom.shareId = shareId;
    callback();
  });

  // Query hook: Pass auth context to queries
  shareDB.use('query', (context, callback) => {
    context.options = {
      ...context.options,
      cookie: context.agent.custom.cookie,
      shareId: context.agent.custom.shareId
    };
    callback();
  });
};
```

### 3.5 Operation Types

```typescript
enum RawOpType {
  Create = 'create',   // Document created
  Del = 'del',         // Document deleted
  Edit = 'edit'        // Document modified (JSON0 ops)
}

interface IRawOp {
  c: string;           // Collection
  d: string;           // Document ID
  v: number;           // Version
  src: string;         // Source client ID
  seq: number;         // Sequence number
  op?: IOtOperation[]; // JSON0 operations
  create?: { type, data };
  del?: true;
}
```

---

## 4. Event Broadcasting

### 4.1 PubSub System

**Two Implementations:**

1. **In-Memory PubSub** (Single instance):
   - Default for development
   - Uses Node.js EventEmitter

2. **Redis PubSub** (Multi-instance):
   - Required for production clusters
   - Configured via `BACKEND_CACHE_REDIS_URI`

**Redis PubSub Implementation** (`share-db/sharedb-redis.pubsub.ts`):

```typescript
export class RedisPubSub extends PubSub {
  client: Redis;      // For publishing
  observer: Redis;    // For subscribing (separate connection)

  async _publish(channels: string[], data: unknown, callback) {
    const message = JSON.stringify(data);
    // Lua script for atomic multi-channel publishing
    this.client.eval(PUBLISH_SCRIPT, 0, message, ...channels);
  }

  async _subscribe(channel: string, callback) {
    await this.observer.subscribe(channel, (err) => {
      callback(err);
    });
  }
}
```

### 4.2 Channel Organization

**Channel Patterns:**

| Pattern | Purpose | Subscribers |
|---------|---------|-------------|
| `{docType}_{tableId}` | All documents in collection | All users viewing table |
| `{docType}_{tableId}.{docId}` | Specific document | Users editing document |

**Example Broadcasting:**
```typescript
// User updates a record
this.pubsub.publish([
  `r_tbl_abc123`,           // Notify all record subscribers
  `r_tbl_abc123.rec_xyz`    // Notify specific record subscribers
], operation, callback);
```

### 4.3 Operation Publishing Flow

```typescript
async publishOpsMap(rawOpMaps: IRawOpMap[]) {
  for (const rawOpMap of rawOpMaps) {
    for (const collection in rawOpMap) {
      for (const docId in rawOpMap[collection]) {
        const rawOp = rawOpMap[collection][docId];

        // Primary channels
        const channels = [
          collection,                    // Collection-level
          `${collection}.${docId}`       // Document-level
        ];

        // Repair attachment metadata
        const repairedOp = await this.repairAttachmentOp(rawOp);

        // Publish to all channels
        this.pubsub.publish(channels, repairedOp, noop);

        // Publish to related channels (e.g., view changes affect records)
        if (this.shouldPublishAction(repairedOp)) {
          this.publishRelatedChannels(tableId, repairedOp);
        }
      }
    }
  }
}
```

---

## 5. Event Emitter System

### 5.1 Module Structure

**Location**: `apps/nestjs-backend/src/event-emitter/`

```typescript
@Module({
  imports: [EventEmitterModule.forRoot(), ShareDbModule, NotificationModule],
  providers: [
    EventEmitterService,
    ActionTriggerListener,           // Automation triggers
    CollaboratorNotificationListener, // Collaboration notifications
    AttachmentListener,              // Attachment processing
    BasePermissionUpdateListener,    // Permission changes
    PinListener,                     // Pin/unpin events
    RecordHistoryListener,           // Audit trail
    TrashListener,                   // Trash operations
  ],
  exports: [EventEmitterService],
})
export class EventEmitterModule {}
```

### 5.2 Domain Events

```typescript
enum Events {
  // Table events
  TABLE_CREATE = 'table.create',
  TABLE_DELETE = 'table.delete',
  TABLE_UPDATE = 'table.update',

  // Field events
  TABLE_FIELD_CREATE = 'table.field.create',
  TABLE_FIELD_DELETE = 'table.field.delete',
  TABLE_FIELD_UPDATE = 'table.field.update',

  // View events
  TABLE_VIEW_CREATE = 'table.view.create',
  TABLE_VIEW_DELETE = 'table.view.delete',
  TABLE_VIEW_UPDATE = 'table.view.update',

  // Record events
  TABLE_RECORD_CREATE = 'table.record.create',
  TABLE_RECORD_DELETE = 'table.record.delete',
  TABLE_RECORD_UPDATE = 'table.record.update',

  // User events
  USER_SIGNOUT = 'user.signout',
  USER_SIGNUP = 'user.signup',
}
```

### 5.3 Event Service

**Converting Operations to Events:**

```typescript
@Injectable()
export class EventEmitterService {
  async ops2Event(rawOpMaps?: IRawOpMap[]): Promise<void> {
    // Collect events from raw operations
    const generatedEvents = this.collectEventsFromRawOpMap(rawOpMaps);

    // Group by table and event type
    const groupedEvents = this.groupEvents(generatedEvents);

    // Aggregate related events
    const aggregatedEvents = this.aggregateEvents(groupedEvents);

    // Emit to appropriate listeners
    for (const event of aggregatedEvents) {
      this.eventEmitter.emit(event.type, event.payload);
    }
  }
}
```

### 5.4 Event Listeners

**Example: Record History Listener**

```typescript
@Injectable()
export class RecordHistoryListener {
  @OnEvent(Events.TABLE_RECORD_UPDATE)
  async handleRecordUpdate(payload: IRecordUpdateEvent) {
    // Create audit trail entry
    await this.recordHistoryService.createHistory({
      tableId: payload.tableId,
      recordId: payload.recordId,
      fieldId: payload.fieldId,
      before: payload.oldValue,
      after: payload.newValue,
      userId: payload.userId,
    });
  }
}
```

**Example: Notification Listener**

```typescript
@Injectable()
export class CollaboratorNotificationListener {
  @OnEvent(Events.TABLE_RECORD_UPDATE)
  async handleRecordUpdate(payload: IRecordUpdateEvent) {
    // Check if user should be notified (e.g., mentioned, assigned)
    const subscribers = await this.getSubscribers(payload);

    for (const userId of subscribers) {
      await this.notificationService.send({
        userId,
        type: 'record_update',
        payload,
      });
    }
  }
}
```

---

## 6. Scalability Considerations

### 6.1 Multi-Instance Setup

For horizontal scaling, ensure:

1. **Redis PubSub**: All instances share same Redis
2. **Sticky Sessions**: Not required (stateless design)
3. **Shared Cache**: Redis for session and cache storage

**Docker Compose Example:**
```yaml
services:
  teable-1:
    environment:
      - BACKEND_CACHE_REDIS_URI=redis://redis:6379

  teable-2:
    environment:
      - BACKEND_CACHE_REDIS_URI=redis://redis:6379

  redis:
    image: redis:7-alpine
```

### 6.2 Connection Management

**Graceful Shutdown:**

```typescript
async onModuleDestroy() {
  // 1. Close WebSocket connections
  this.server?.clients.forEach((client) => client.terminate());

  // 2. Close ShareDB
  await new Promise((resolve, reject) => {
    this.shareDb.close((err) => {
      if (err) reject(err);
      else resolve(null);
    });
  });

  // 3. Close WebSocket server
  await new Promise((resolve) => {
    this.server?.close(() => resolve(null));
  });
}
```

### 6.3 Performance Features

| Feature | Implementation |
|---------|----------------|
| **Message Compression** | `perMessageDeflate: true` |
| **Operation Retries** | `maxSubmitRetries: 3` |
| **Async Persistence** | Transaction hooks trigger pubsub |
| **Presence Tracking** | ShareDB presence middleware |
| **Query Projection** | Fetch only needed fields |

### 6.4 Load Balancing

**Recommended Configuration:**

```nginx
upstream teable_ws {
  ip_hash;  # Optional: sticky sessions
  server teable-1:3000;
  server teable-2:3000;
}

server {
  location /socket {
    proxy_pass http://teable_ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;
  }
}
```

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| WebSocket Gateway | `apps/nestjs-backend/src/ws/ws.gateway.ts` |
| Dev Gateway | `apps/nestjs-backend/src/ws/ws.gateway.dev.ts` |
| ShareDB Service | `apps/nestjs-backend/src/share-db/share-db.service.ts` |
| ShareDB Adapter | `apps/nestjs-backend/src/share-db/share-db.adapter.ts` |
| Redis PubSub | `apps/nestjs-backend/src/share-db/sharedb-redis.pubsub.ts` |
| Auth Middleware | `apps/nestjs-backend/src/share-db/auth.middleware.ts` |
| Event Emitter | `apps/nestjs-backend/src/event-emitter/event-emitter.service.ts` |
| Event Listeners | `apps/nestjs-backend/src/event-emitter/listeners/` |

---

## Real-time Data Flow

```
User Action (Client)
    │
    ▼
WebSocket Message
    │
    ▼
ShareDB Submit Operation
    │
    ├──────────────────────────────────┐
    ▼                                  ▼
Database Transaction            PubSub Publish
    │                                  │
    ▼                                  ▼
Prisma Middleware             Redis Channels
    │                                  │
    ▼                                  │
CLS: Collect Cache Keys               │
    │                                  │
    ▼                                  │
Transaction Commit                    │
    │                                  │
    ├──────────────────────────────────┤
    ▼                                  ▼
Clear Cache Keys              Broadcast to Clients
    │                                  │
    ▼                                  ▼
Event Emitter             Client Apply Operation
    │                                  │
    ▼                                  ▼
Side Effects               UI Update
(notifications, history, etc.)
```

---

*See also: [Main Architecture](../ANALYSIS.md) | [Database Architecture](./ARCHITECTURE-DATABASE.md)*
