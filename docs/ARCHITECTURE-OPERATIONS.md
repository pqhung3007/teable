# Teable Operations Architecture

> **Detailed documentation of job queues, caching, import/export, and file storage.**

## Table of Contents

1. [Background Job Processing](#1-background-job-processing)
2. [Caching Strategy](#2-caching-strategy)
3. [Import Pipeline](#3-import-pipeline)
4. [Export Pipeline](#4-export-pipeline)
5. [File Storage](#5-file-storage)

---

## 1. Background Job Processing

### 1.1 BullMQ Configuration

**Location**: `apps/nestjs-backend/src/app.module.ts`

```typescript
// Conditional BullMQ setup (requires Redis)
ConditionalModule.registerWhen(
  BullModule.forRootAsync({
    useFactory: async (configService: ConfigService) => {
      const redisUri = configService.get<ICacheConfig>('cache')?.redis.uri;
      const redis = new Redis(redisUri, {
        lazyConnect: true,
        maxRetriesPerRequest: null
      });
      await redis.connect();
      return { connection: redis };
    }
  }),
  (env) => Boolean(env.BACKEND_CACHE_REDIS_URI)
)
```

### 1.2 Queue Definitions

| Queue Name | Processor | Purpose | Concurrency |
|------------|-----------|---------|-------------|
| `attachments-crop-queue` | AttachmentsCropQueueProcessor | Image thumbnail generation | 1 |
| `mailSenderQueue` | MailSenderMergeProcessor | Email notifications | 1 |
| `base-import-attachments-queue` | BaseImportAttachmentsQueueProcessor | Import attachment files | 1 |
| `base-import-csv-queue` | BaseImportCsvQueueProcessor | Import table CSV data | 1 |
| `import-table-csv-queue` | ImportTableCsvQueueProcessor | Import records | 1 |
| `import-table-csv-chunk-queue` | ImportTableCsvChunkQueueProcessor | Parse large CSV files | 6 |

### 1.3 Job Options

```typescript
const queueOptions = {
  removeOnComplete: { count: 2000 },  // Keep last 2000 completed jobs
  removeOnFail: { count: 5000 },      // Keep last 5000 failed jobs
};

// Per-job configuration
await queue.add('jobName', data, {
  delay: 1000,                // Delay in milliseconds
  jobId: `unique_${id}`,      // Prevent duplicates
  removeOnComplete: true,     // Remove immediately
  removeOnFail: true          // Remove on failure
});
```

### 1.4 Fallback System (No Redis)

When Redis is unavailable, jobs process in-memory:

```typescript
// Fallback uses Node.js EventEmitter
export class FallbackQueueService {
  private emitter = new EventEmitter();

  add(name: string, data: any) {
    this.emitter.emit(name, { data });
  }

  process(name: string, handler: (job) => Promise<void>) {
    this.emitter.on(name, handler);
  }
}
```

### 1.5 Job Processing Pattern

```typescript
@Processor(QUEUE_NAME)
export class MyQueueProcessor extends WorkerHost {
  private processedJobs = new Set<string>();  // Idempotency

  async process(job: Job) {
    // Prevent duplicate processing
    if (this.processedJobs.has(job.id)) {
      return;
    }
    this.processedJobs.add(job.id);

    try {
      await this.doWork(job.data);
    } catch (error) {
      this.logger.error(`Job ${job.id} failed`, error);
      throw error;  // BullMQ handles retry
    }
  }

  @OnWorkerEvent('completed')
  async onCompleted(job: Job) {
    // Trigger next job in pipeline
    await this.nextQueue.add('nextJob', { ...job.data });
  }

  @OnWorkerEvent('error')
  async onError(job: Job) {
    // Cleanup related jobs
    const allJobs = await this.queue.getJobs(['waiting', 'active']);
    for (const relatedJob of allJobs) {
      await relatedJob.remove();
    }
  }
}
```

---

## 2. Caching Strategy

### 2.1 Dual-Tier Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Tier 1: Application Cache          │
│  ┌───────────────────────────────────────────────┐ │
│  │ CacheService (Keyv)                           │ │
│  │ • Backends: Memory | SQLite | Redis           │ │
│  │ • Sessions, tokens, temp data                 │ │
│  │ • TTL with jitter (cache stampede prevention) │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│             Tier 2: Performance Cache (Optional)    │
│  ┌───────────────────────────────────────────────┐ │
│  │ PerformanceCacheService                       │ │
│  │ • Redis only                                  │ │
│  │ • Query results, entity caching               │ │
│  │ • Redlock for concurrent access               │ │
│  │ • Statistics and monitoring                   │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 2.2 Application Cache (Tier 1)

**Configuration:**
```bash
BACKEND_CACHE_PROVIDER=redis   # memory | sqlite | redis
BACKEND_CACHE_REDIS_URI=redis://localhost:6379
BACKEND_CACHE_SQLITE_URI=sqlite://.assets/.cache.db
```

**Cache Keys:**
```typescript
// Authentication
auth:session-store:{sid}
auth:session-user:{userId}
auth:session-expire:{sid}

// Attachments
attachment:signature:{token}
attachment:upload:{token}
attachment:preview:{token}

// Operations
operations:undo:{userId}:{tableId}:{windowId}
operations:redo:{userId}:{tableId}:{windowId}

// Rate Limiting
signin:attempts:{email}
signin:lockout:{email}
```

**TTL with Jitter:**
```typescript
async set(key, value, ttl) {
  // Add 20-60 seconds jitter to prevent cache stampede
  const jitter = getRandomInt(20, 60);
  const finalTtl = ttl + jitter;
  await this.keyv.set(key, value, finalTtl * 1000);
}
```

### 2.3 Performance Cache (Tier 2)

**Configuration:**
```bash
BACKEND_PERFORMANCE_CACHE=redis://localhost:6379
```

**Cache Keys:**
```typescript
// Entity caching
user:{userId}
collaborator:{resourceId}
access-token:{tokenId}
instance:setting

// Query caching
record:{path}:{tableId}:{version}:{queryHash}
agg:{path}:{tableId}:{version}:{queryHash}
```

**Decorator-Based Caching:**
```typescript
@PerformanceCache({
  ttl: 30,                              // Seconds
  keyGenerator: generateUserCacheKey,
  statsType: 'user',
  preventConcurrent: false
})
async getUserById(id: string) {
  return await this.prisma.user.findUnique({ where: { id } });
}
```

### 2.4 Cache Invalidation

**Prisma Middleware Pattern:**
```typescript
// Automatically clear cache on model updates
this.prisma.$use(async (params, next) => {
  const clearCacheKeys = [];

  if (params.model === 'User' && params.action.includes('update')) {
    clearCacheKeys.push(`user:${params.args.where.id}`);
  }

  // Non-transactional: Clear immediately
  if (!params.runInTransaction) {
    await Promise.all(clearCacheKeys.map(key => cache.del(key)));
  } else {
    // Transactional: Defer to CLS for post-commit clearing
    cls.set('clearCacheKeys', [...existing, ...clearCacheKeys]);
  }

  return next(params);
});
```

### 2.5 Distributed Locking (Redlock)

```typescript
async wrap(key, fn, options) {
  const cached = await this.get(key);
  if (cached) return cached;

  if (options.preventConcurrent) {
    // Acquire distributed lock
    return await this.redlock.using([`lock:${key}`], 10000, async () => {
      // Double-check after acquiring lock
      const cachedAfterLock = await this.get(key);
      if (cachedAfterLock) return cachedAfterLock;

      const result = await fn();
      await this.set(key, result, options.ttl);
      return result;
    });
  }

  const result = await fn();
  await this.set(key, result, options.ttl);
  return result;
}
```

---

## 3. Import Pipeline

### 3.1 Import Flow Overview

```
File Upload
    │
    ▼
┌───────────────────────────────────────┐
│ ImportOpenApiService.analyze()        │
│ • Parse file headers                  │
│ • Infer column types                  │
│ • Generate field configurations       │
└────────────────┬──────────────────────┘
                 │
                 ▼
┌───────────────────────────────────────┐
│ createTableFromImport()               │
│ • Create table with inferred schema   │
│ • Enqueue import job                  │
└────────────────┬──────────────────────┘
                 │
                 ▼
┌───────────────────────────────────────┐
│ ImportTableCsvChunkQueueProcessor     │
│ • Worker thread CSV parsing           │
│ • Chunk data to storage               │
│ • Enqueue chunk import jobs           │
└────────────────┬──────────────────────┘
                 │
                 ▼
┌───────────────────────────────────────┐
│ ImportTableCsvQueueProcessor          │
│ • Read chunk from storage             │
│ • Type-cast values                    │
│ • Bulk insert records                 │
└───────────────────────────────────────┘
```

### 3.2 Schema Inference

**Location**: `apps/nestjs-backend/src/features/import/open-api/import.class.ts`

```typescript
// Type detection priority
SUPPORTED_TYPES = [
  FieldType.Checkbox,      // "true"/"false" values
  FieldType.Number,        // Numeric values
  FieldType.Date,          // ISO dates
  FieldType.LongText,      // Contains newlines
  FieldType.SingleLineText // Default fallback
];

// Scan first 500 rows (CHECK_LINES)
for (const row of rows.slice(0, 500)) {
  for (const [index, value] of row.entries()) {
    // Test against Zod schemas
    if (!checkboxSchema.safeParse(value).success) {
      columnTypes[index].delete(FieldType.Checkbox);
    }
    if (!numberSchema.safeParse(value).success) {
      columnTypes[index].delete(FieldType.Number);
    }
    // ... etc
  }
}
```

### 3.3 Chunked Processing

**CSV Parsing Configuration:**
```typescript
CHUNK_SIZE = 1024 * 1024 * 0.2;  // 200 KB
MAX_CHUNK_LENGTH = 500;          // 500 rows

// Stream processing with PapaParse
Papa.parse(stream, {
  chunk: async (results, parser) => {
    buffer.push(...results.data);

    if (buffer.length >= MAX_CHUNK_LENGTH ||
        sizeof(buffer) > CHUNK_SIZE) {
      parser.pause();
      await processChunk(buffer);
      buffer = [];
      parser.resume();
    }
  }
});
```

**Worker Thread Parsing:**
```typescript
// Parse in worker thread to avoid blocking
const worker = new Worker('./parse.worker.js');
worker.postMessage({ filePath, options });
worker.on('message', (chunk) => {
  // Store chunk to temporary storage
  await storage.uploadFile(bucket, chunkPath, chunk);
  // Enqueue insert job
  await queue.add('insertChunk', { chunkPath });
});
```

### 3.4 Batch Insertion

```typescript
// Insert in configurable chunks
const chunkSize = calcChunkSize(recordCount);  // Usually 1000

for (let i = 0; i < records.length; i += chunkSize) {
  const chunk = records.slice(i, i + chunkSize);

  // For new tables: Direct SQL insert (fastest)
  await createRecordsOnlySql(tableId, chunk);

  // For existing tables: Full validation
  await multipleCreateRecords(tableId, chunk);
}
```

### 3.5 Progress Tracking

```typescript
// ShareDB presence for real-time updates
const presence = shareDb.connect().getPresence(channel);
presence.submit({ loading: true, progress: 0 });

// Update progress during import
for (let i = 0; i < chunks.length; i++) {
  await processChunk(chunks[i]);
  presence.submit({ progress: (i + 1) / chunks.length * 100 });
}

// Completion notification
await notificationService.sendImportResultNotify({
  baseId,
  tableId,
  message: `🎉 ${tableName} imported successfully`
});
```

---

## 4. Export Pipeline

### 4.1 CSV Export

**Location**: `apps/nestjs-backend/src/features/export/open-api/export-open-api.service.ts`

```typescript
async exportCsvFromTable(response: Response, tableId: string, viewId?: string) {
  // Set streaming headers
  response.setHeader('Content-Type', 'text/csv; charset=utf-8');
  response.setHeader('Content-Disposition', `attachment; filename=${fileName}.csv`);

  // Create readable stream
  const csvStream = new Readable({ read() {} });
  csvStream.pipe(response);

  // Write UTF-8 BOM for Excel compatibility
  csvStream.push('\uFEFF');

  // Write headers
  csvStream.push(Papa.unparse([headers.map(h => h.name)]));

  // Stream records in batches
  let skip = 0;
  const take = 1000;

  while (true) {
    const { records } = await recordService.getRecords(tableId, {
      take, skip, viewId
    });

    if (records.length === 0) break;

    // Format cell values based on field type
    const csvData = Papa.unparse(records.map(formatRecord));
    csvStream.push('\r\n' + csvData);

    skip += take;
  }

  csvStream.push(null);  // End stream
}
```

### 4.2 Field-Type Formatting

```typescript
function formatCellValue(value: unknown, field: FieldInstance): string {
  switch (field.type) {
    case FieldType.Attachment:
      // "name presignedUrl, name presignedUrl"
      return value.map(v => `${v.name} ${v.presignedUrl}`).join(',');

    case FieldType.Link:
      // Extract linked record titles
      return value.map(v => v.title).join(', ');

    case FieldType.MultipleSelect:
      return value.join(', ');

    case FieldType.Date:
      return dayjs(value).format(field.options.dateFormat);

    default:
      return field.cellValue2String(value);
  }
}
```

### 4.3 View-Specific Export

```typescript
// Apply view filters and hidden fields
const exportQuery = {
  // Merge view filter with custom filter
  filter: mergeFilters(view.filter, customFilter),

  // Exclude hidden fields
  projection: fields
    .filter(f => !view.options.hiddenFieldIds?.includes(f.id))
    .map(f => f.id)
};
```

---

## 5. File Storage

### 5.1 Storage Providers

| Provider | Use Case | Configuration |
|----------|----------|---------------|
| **Local** | Development | `BACKEND_STORAGE_LOCAL_PATH=.assets/uploads` |
| **MinIO** | Self-hosted S3 | `BACKEND_STORAGE_MINIO_ENDPOINT` |
| **S3** | AWS deployment | `BACKEND_STORAGE_S3_REGION` |
| **Aliyun** | China deployment | `BACKEND_STORAGE_S3_ENDPOINT` (Aliyun OSS) |

### 5.2 Upload Flow

```
┌─────────────────────────────────────────────────────┐
│ 1. Client requests presigned URL                    │
│    POST /api/attachments/signature                  │
│    → Returns: presignedUrl, token, path             │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ 2. Client uploads directly to storage               │
│    PUT presignedUrl (S3/MinIO)                      │
│    OR PUT /api/attachments/upload/:token (Local)    │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ 3. Client notifies completion                       │
│    POST /api/attachments/notify/:token              │
│    → Creates attachment record                      │
│    → Queues thumbnail generation                    │
│    → Returns attachment metadata with preview URL   │
└─────────────────────────────────────────────────────┘
```

### 5.3 Storage Adapter Interface

```typescript
interface IStorageAdapter {
  // File operations
  uploadFile(bucket: string, path: string, stream: Readable): Promise<void>;
  getFileStream(bucket: string, path: string): Promise<Readable>;
  deleteFile(bucket: string, path: string): Promise<void>;

  // URL generation
  getPreviewUrl(bucket: string, path: string, options?): Promise<string>;
  getPresignedUrl(bucket: string, path: string, options): Promise<IPresignResult>;

  // Metadata
  getFileMeta(bucket: string, path: string): Promise<IFileMeta>;
}
```

### 5.4 Bucket Organization

| Bucket | Access | Content |
|--------|--------|---------|
| **public** | Public URLs | Avatars, logos, templates |
| **private** | Signed URLs | Table attachments, exports, imports |

### 5.5 Thumbnail Generation

```typescript
@Processor('attachments-crop-queue')
export class AttachmentsCropQueueProcessor extends WorkerHost {
  async process(job: Job<IRecordImageJob>) {
    const { bucket, path, token, height } = job.data;

    // Skip if original is smaller than target
    if (height && height <= THUMBNAIL_HEIGHT) {
      return;
    }

    // Generate thumbnails
    const smPath = await this.cropImage(path, SM_HEIGHT);  // 56px
    const lgPath = await this.cropImage(path, LG_HEIGHT);  // 525px

    // Update attachment record
    await this.prisma.attachments.update({
      where: { token },
      data: {
        thumbnailPath: JSON.stringify({ sm: smPath, lg: lgPath })
      }
    });
  }

  async cropImage(sourcePath: string, targetHeight: number) {
    const image = await sharp(await this.storage.getFileStream(sourcePath));
    const resized = await image.resize({ height: targetHeight }).toBuffer();
    const newPath = generateThumbnailPath(sourcePath, targetHeight);
    await this.storage.uploadFile(bucket, newPath, resized);
    return newPath;
  }
}
```

### 5.6 Signed URL Security

```typescript
// Token encryption for private files
const ALGORITHM = 'aes-128-cbc';
const KEY = process.env.BACKEND_STORAGE_ENCRYPTION_KEY;
const IV = process.env.BACKEND_STORAGE_ENCRYPTION_IV;

function encryptToken(data: string): string {
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, IV);
  return cipher.update(data, 'utf8', 'base64') + cipher.final('base64');
}

function generateSignedUrl(path: string): string {
  const token = encryptToken(JSON.stringify({ path, exp: Date.now() + TTL }));
  return `/api/attachments/read/${path}?token=${token}`;
}
```

---

## Key Configuration Summary

### Environment Variables

```bash
# Job Queue (Redis required)
BACKEND_CACHE_REDIS_URI=redis://localhost:6379

# Caching
BACKEND_CACHE_PROVIDER=redis
BACKEND_PERFORMANCE_CACHE=redis://localhost:6379

# Storage
BACKEND_STORAGE_PROVIDER=minio
BACKEND_STORAGE_PUBLIC_BUCKET=public
BACKEND_STORAGE_PRIVATE_BUCKET=private
BACKEND_STORAGE_TOKEN_EXPIRE_IN=6d

# MinIO
BACKEND_STORAGE_MINIO_ENDPOINT=minio.local
BACKEND_STORAGE_MINIO_PORT=9000
BACKEND_STORAGE_MINIO_ACCESS_KEY=minioadmin
BACKEND_STORAGE_MINIO_SECRET_KEY=minioadmin

# S3
BACKEND_STORAGE_S3_REGION=us-east-1
BACKEND_STORAGE_S3_ACCESS_KEY=...
BACKEND_STORAGE_S3_SECRET_KEY=...

# Import limits
MAX_ATTACHMENT_UPLOAD_SIZE=52428800  # 50 MB
MAX_OPENAPI_ATTACHMENT_UPLOAD_SIZE=52428800
```

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| BullMQ Setup | `apps/nestjs-backend/src/app.module.ts` |
| Queue Module | `apps/nestjs-backend/src/event-emitter/event-job/` |
| Cache Service | `apps/nestjs-backend/src/cache/cache.service.ts` |
| Performance Cache | `apps/nestjs-backend/src/performance-cache/` |
| Import Service | `apps/nestjs-backend/src/features/import/` |
| Export Service | `apps/nestjs-backend/src/features/export/` |
| Attachments | `apps/nestjs-backend/src/features/attachments/` |
| Storage Plugins | `apps/nestjs-backend/src/features/attachments/plugins/` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Database Architecture](./ARCHITECTURE-DATABASE.md)*
