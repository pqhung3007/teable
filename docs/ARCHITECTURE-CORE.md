# Teable Core Backend Architecture Decisions

> **Detailed documentation of data persistence, query engine, views, API layer, and plugin system.**

## Table of Contents

1. [Data Persistence Architecture](#1-data-persistence-architecture)
2. [Record Storage](#2-record-storage)
3. [Relationship Management](#3-relationship-management)
4. [View Processing Backend](#4-view-processing-backend)
5. [API Layer Architecture](#5-api-layer-architecture)
6. [Search & Indexing](#6-search--indexing)
7. [Plugin/Extension Backend](#7-pluginextension-backend)

---

## 1. Data Persistence Architecture

### 1.1 Dynamic Schema Management

Teable uses a **hybrid approach** combining metadata tables with dynamically created physical tables:

```
┌─────────────────────────────────────────────────────────────┐
│                    Metadata Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  TableMeta  │  │    Field    │  │        View         │ │
│  │  - id       │  │  - id       │  │  - id               │ │
│  │  - dbTable  │  │  - dbField  │  │  - filter/sort/group│ │
│  │  - version  │  │  - type     │  │  - columnMeta       │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Physical Layer                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  tbl_abc123 (Physical Table)                            ││
│  │  ┌─────────┬─────────┬─────────┬─────────┬───────────┐ ││
│  │  │ __id    │ fld_001 │ fld_002 │ fld_003 │ __version │ ││
│  │  │ VARCHAR │ VARCHAR │ NUMERIC │ JSONB   │ INTEGER   │ ││
│  │  └─────────┴─────────┴─────────┴─────────┴───────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

| Approach | Description | Why |
|----------|-------------|-----|
| **One Physical Table per Logical Table** | Each user table maps to a real database table | Query performance, native DB features |
| **Dynamic Column Creation** | Fields create columns via DDL | Type safety, indexing support |
| **JSONB for Complex Types** | Attachments, links, multi-select use JSONB | Flexibility with queryability |
| **Metadata-Data Separation** | Schema info in metadata tables | Supports schema versioning |

### 1.2 Field Type System

**20+ Field Types Supported:**

| Category | Field Types | Storage Type |
|----------|-------------|--------------|
| **Text** | SingleLineText, LongText | VARCHAR, TEXT |
| **Numeric** | Number, Rating, AutoNumber | NUMERIC, INTEGER, SERIAL |
| **Selection** | SingleSelect, MultipleSelect | VARCHAR, JSONB |
| **Date/Time** | Date, CreatedTime, LastModifiedTime | TIMESTAMP |
| **Boolean** | Checkbox | BOOLEAN |
| **Files** | Attachment | JSONB |
| **Relations** | Link, User | JSONB |
| **Computed** | Formula, Lookup, Rollup | GENERATED or VIEW |
| **System** | CreatedBy, LastModifiedBy | VARCHAR |
| **Interactive** | Button | N/A |

### 1.3 Field Metadata Storage

```typescript
// Field model stores configuration as JSON
interface IFieldMetadata {
  options: {          // Field-type-specific options
    expression?: string;      // Formula expression
    choices?: ISelectOption[]; // Select options
    dateFormat?: string;      // Date formatting
    precision?: number;       // Number precision
    // ... type-specific options
  };
  lookupOptions?: {   // For lookup/rollup fields
    foreignTableId: string;
    lookupFieldId: string;
    filter?: IFilter;
    rollupFunction?: string;
  };
  aiConfig?: {        // AI field configuration
    model: string;
    prompt: string;
  };
  meta?: {            // Additional metadata
    persistedAsGeneratedColumn?: boolean;
  };
}
```

### 1.4 Schema Versioning

**Version Tracking at Multiple Levels:**

```prisma
model TableMeta {
  version Int @default(1)  // Incremented on schema changes
}

model Field {
  version Int @default(1)  // Incremented on field updates
}

model Ops {
  version Int              // Operation versioning for OT
  @@unique([collection, docId, version])
}
```

**Version Uses:**
- Optimistic locking for concurrent updates
- ShareDB operational transformation
- Cache invalidation triggers

---

## 2. Record Storage

### 2.1 Row-Level Data Structure

Each record in a physical table has:

```sql
CREATE TABLE "tbl_xxx" (
  -- System columns (always present)
  "__id" VARCHAR PRIMARY KEY,            -- Record ID (rec_xxx)
  "__auto_number" SERIAL,                -- Auto-increment
  "__created_time" TIMESTAMP,            -- Creation timestamp
  "__last_modified_time" TIMESTAMP,      -- Last modification
  "__created_by" VARCHAR,                -- Creator user ID
  "__last_modified_by" VARCHAR,          -- Last modifier
  "__version" INTEGER DEFAULT 1,         -- OT version

  -- User-defined columns (dynamic)
  "fld_abc123" VARCHAR,                  -- Text field
  "fld_def456" NUMERIC,                  -- Number field
  "fld_ghi789" JSONB,                    -- Multi-value field
  "fld_formula" NUMERIC GENERATED ALWAYS AS (...) STORED
);
```

### 2.2 Cell Value Storage Strategy

| Field Type | Storage | Example Value |
|------------|---------|---------------|
| SingleLineText | VARCHAR | `'Hello World'` |
| Number | NUMERIC | `123.45` |
| Checkbox | BOOLEAN | `true` |
| SingleSelect | VARCHAR | `'Option A'` |
| MultipleSelect | JSONB | `['Option A', 'Option B']` |
| Date | TIMESTAMP | `'2024-01-15T10:30:00Z'` |
| Attachment | JSONB | `[{id, name, size, token, path}]` |
| Link | JSONB | `[{id: 'rec_xxx', title: '...'}]` |
| User | JSONB | `[{id: 'usr_xxx', title: '...'}]` |

### 2.3 Audit Trail Implementation

**Record History Tracking:**

```prisma
model RecordHistory {
  id          String   @id
  tableId     String
  recordId    String
  fieldId     String
  before      String?  // JSON: previous value
  after       String?  // JSON: new value
  createdTime DateTime @default(now())
  createdBy   String

  @@index([tableId, recordId, createdTime])
}
```

**Event-Driven History:**
```typescript
@OnEvent(Events.TABLE_RECORD_UPDATE, { async: true })
async recordUpdateListener(event: RecordUpdateEvent) {
  await this.recordHistoryService.createHistory({
    tableId: event.payload.tableId,
    recordId: event.payload.recordId,
    fieldId: event.payload.fieldId,
    before: event.payload.oldValue,
    after: event.payload.newValue,
    createdBy: event.context.userId
  });
}
```

### 2.4 Soft Delete & Trash

```prisma
model TableTrash {
  id          String   @id
  tableId     String
  snapshot    String   // JSON: full table snapshot
  createdTime DateTime
  createdBy   String
}

model RecordTrash {
  id          String   @id
  tableId     String
  recordId    String
  snapshot    String   // JSON: record snapshot
  createdTime DateTime
  createdBy   String
}
```

**Trash Operations:**
- Records/tables moved to trash on delete
- Configurable retention period
- Restore capabilities via TrashService

---

## 3. Relationship Management

### 3.1 Link Fields Between Tables

**Bidirectional Link System:**

```
Table A                          Table B
┌─────────────────┐              ┌─────────────────┐
│ fld_link_to_B   │─────────────▶│ fld_link_to_A   │
│ (Link field)    │◀─────────────│ (Symmetric link)│
└─────────────────┘              └─────────────────┘
```

**Link Field Storage:**
```typescript
// Link field options
interface ILinkFieldOptions {
  foreignTableId: string;      // Target table
  relationship: 'oneToOne' | 'oneToMany' | 'manyToMany';
  isOneWay: boolean;           // Symmetric or one-way
  symmetricFieldId?: string;   // Paired field in target table
  dbForeignKeyName?: string;   // FK constraint name
}

// Cell value storage
[
  { id: 'rec_target1', title: 'Record 1' },
  { id: 'rec_target2', title: 'Record 2' }
]
```

### 3.2 Foreign Key Handling

**Junction Table for Many-to-Many:**

```sql
-- Auto-created for manyToMany relationships
CREATE TABLE "junction_tblA_tblB" (
  "fld_linkA" VARCHAR REFERENCES "tbl_A"("__id"),
  "fld_linkB" VARCHAR REFERENCES "tbl_B"("__id"),
  PRIMARY KEY ("fld_linkA", "fld_linkB")
);
```

**LinkService Responsibilities:**
- Create symmetric link fields
- Maintain junction tables
- Cascade link value updates
- Handle link field deletion

### 3.3 Cascading Operations

```typescript
// Link deletion handling
async deleteField(fieldId: string) {
  const field = await this.getField(fieldId);

  if (field.type === FieldType.Link) {
    // Delete symmetric field
    if (field.options.symmetricFieldId) {
      await this.deleteField(field.options.symmetricFieldId);
    }
    // Drop junction table if exists
    await this.dropJunctionTable(field);
  }
}
```

### 3.4 Reference Tracking

**Dependency Graph:**

```prisma
model Reference {
  id          String @id
  toFieldId   String   // Dependent field
  fromFieldId String   // Source field
  createdTime DateTime

  @@unique([toFieldId, fromFieldId])
  @@index([fromFieldId])
}
```

**Used For:**
- Formula field dependencies
- Lookup/Rollup source tracking
- Circular reference detection
- Cascading recalculation

---

## 4. View Processing Backend

### 4.1 View Types

| Type | Purpose | Key Options |
|------|---------|-------------|
| **Grid** | Spreadsheet-like | `rowHeight`, `frozenColumnCount` |
| **Kanban** | Card-based boards | `stackFieldId`, `coverFieldId` |
| **Gallery** | Visual grid | `coverFieldId`, `isCoverFit` |
| **Calendar** | Date-based | `startDateFieldId`, `endDateFieldId` |
| **Form** | Data entry | `coverUrl`, `logoUrl`, `submitLabel` |
| **Plugin** | Custom views | `pluginId`, `pluginInstallId` |

### 4.2 Server-Side Filtering

**Filter Structure:**
```typescript
interface IFilter {
  conjunction: 'and' | 'or';
  filterSet: (IFilterItem | IFilter)[];  // Nested groups
}

interface IFilterItem {
  fieldId: string;
  operator: FilterOperator;
  value: any;
  isSymbol?: boolean;  // Field reference
}
```

**40+ Filter Operators:**
- Text: `is`, `isNot`, `contains`, `doesNotContain`, `startsWith`, `endsWith`
- Numbers: `isGreater`, `isLess`, `isGreaterEqual`, `isLessEqual`
- Dates: `isBefore`, `isAfter`, `isWithIn`, `today`, `pastWeek`, etc.
- Multi-value: `isAnyOf`, `hasAllOf`, `isExactly`
- Null: `isEmpty`, `isNotEmpty`

### 4.3 Grouping & Aggregation

**Group Configuration:**
```typescript
interface IGroup {
  fieldId: string;
  order: 'asc' | 'desc';
}
// Max 3 groups supported
```

**22 Aggregation Functions:**
```typescript
enum StatisticsFunc {
  Count, Empty, Filled, Unique,
  Max, Min, Sum, Average,
  Checked, UnChecked,
  PercentEmpty, PercentFilled, PercentUnique,
  PercentChecked, PercentUnChecked,
  EarliestDate, LatestDate,
  DateRangeOfDays, DateRangeOfMonths,
  TotalAttachmentSize
}
```

### 4.4 View Configuration Storage

```prisma
model View {
  id         String
  type       String    // grid, kanban, gallery, calendar, form, plugin
  options    String?   // JSON: view-type-specific options
  filter     String?   // JSON: IFilter
  sort       String?   // JSON: ISortItem[]
  group      String?   // JSON: IGroupItem[]
  columnMeta String?   // JSON: field visibility/order/width
  shareMeta  String?   // JSON: sharing configuration
}
```

### 4.5 Column Meta Per View Type

| View Type | Column Properties |
|-----------|-------------------|
| Grid | `width`, `hidden`, `statisticFunc` |
| Kanban | `visible` |
| Gallery | `visible` |
| Calendar | `visible` |
| Form | `visible`, `required` |
| Plugin | `hidden` |

---

## 5. API Layer Architecture

### 5.1 REST API Design

**Resource Naming:**
```
/api/space                     # Workspace management
/api/base                      # Database management
/api/table/{tableId}           # Table operations
/api/table/{tableId}/field     # Field CRUD
/api/table/{tableId}/view      # View CRUD
/api/table/{tableId}/record    # Record CRUD
```

**CRUD Patterns:**
```typescript
GET    /api/table/{tableId}/record          // List records
POST   /api/table/{tableId}/record          // Create records
GET    /api/table/{tableId}/record/{id}     // Get single record
PATCH  /api/table/{tableId}/record/{id}     // Update record
DELETE /api/table/{tableId}/record/{id}     // Delete record
```

### 5.2 Bulk Operations

```typescript
// Bulk create
POST /api/table/{tableId}/record
Body: { records: [{ fields: {...} }, ...], fieldKeyType?: 'id' | 'name' }

// Bulk update
PATCH /api/table/{tableId}/record
Body: { records: [{ id: '...', fields: {...} }, ...] }

// Bulk delete
DELETE /api/table/{tableId}/record
Body: { recordIds: ['rec_xxx', 'rec_yyy'] }
```

### 5.3 Query Language (Filtering)

```typescript
GET /api/table/{tableId}/record?filter={...}&sort={...}

// Filter example
{
  "conjunction": "and",
  "filterSet": [
    { "fieldId": "fld_name", "operator": "contains", "value": "John" },
    { "fieldId": "fld_status", "operator": "isAnyOf", "value": ["Active", "Pending"] }
  ]
}

// Sort example
[
  { "fieldId": "fld_date", "order": "desc" },
  { "fieldId": "fld_name", "order": "asc" }
]
```

### 5.4 Error Response Standards

```typescript
interface IErrorResponse {
  message: string;
  status: number;      // HTTP status code
  code: string;        // Application error code
  data?: {
    errors?: IValidationError[];
  };
}

// Example
{
  "message": "Field not found",
  "status": 404,
  "code": "FIELD_NOT_FOUND"
}
```

### 5.5 Rate Limiting

```typescript
// Configured per endpoint
@UseGuards(ThrottlerGuard)
@Throttle({ default: { limit: 100, ttl: 60000 } })
async createRecords() { ... }
```

### 5.6 API Versioning

- Currently: No explicit versioning (v1 implicit)
- Strategy: URI-based when needed (`/api/v2/...`)
- Deprecation: Via headers and documentation

---

## 6. Search & Indexing

### 6.1 Full-Text Search

**PostgreSQL Full-Text Search:**
```typescript
// Search query construction
searchQuery(qb, searchValue, fields) {
  const searchableFields = fields.filter(f =>
    [FieldType.SingleLineText, FieldType.LongText].includes(f.type)
  );

  qb.where((builder) => {
    searchableFields.forEach((field) => {
      builder.orWhereRaw(
        `${field.dbFieldName}::text ILIKE ?`,
        [`%${searchValue}%`]
      );
    });
  });
}
```

### 6.2 Index Strategy

**Automatic Indexes:**
- Primary key (`__id`)
- Auto-number (`__auto_number`)
- Foreign keys for link fields

**Custom Indexes (via migrations):**
```sql
-- Performance indexes
CREATE INDEX "idx_table_meta_order" ON "table_meta"("order");
CREATE INDEX "idx_field_lookup" ON "field"("lookup_linked_field_id");
CREATE INDEX "idx_collaborator_resource" ON "collaborator"("resource_id");
```

---

## 7. Plugin/Extension Backend

### 7.1 Plugin Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Plugin Host (Teable)                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                   Plugin Bridge (IPC)                    ││
│  │  • expandRecord()    • updateStorage()                  ││
│  │  • getAuthCode()     • getSelectionRecords()           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            ┌───────────────┐   ┌───────────────┐
            │ Chart Plugin  │   │ Form Plugin   │
            │ (Dashboard)   │   │ (View)        │
            └───────────────┘   └───────────────┘
```

### 7.2 Plugin Positions

| Position | Description | Use Case |
|----------|-------------|----------|
| `dashboard` | Dashboard widgets | Charts, metrics |
| `view` | Custom view types | Spreadsheet forms |
| `contextMenu` | Right-click menus | Quick actions |
| `panel` | Side panels | Details, tools |

### 7.3 Plugin Registration

```typescript
interface IPlugin {
  id: string;
  name: string;
  description: string;
  logo: string;
  url: string;                    // Plugin URL
  positions: PluginPosition[];    // Where it can be used
  status: 'developing' | 'reviewing' | 'published';
  secret: string;                 // API secret
  pluginUser?: string;            // System user for data access
}
```

### 7.4 Plugin SDK Bridge

**Parent → Plugin Methods:**
```typescript
interface IParentBridgeMethods {
  expandRecord(recordIds: string[]): void;
  expandPlugin(): void;
  updateStorage(storage: Record<string, unknown>): Promise<Record<string, unknown>>;
  getAuthCode(): Promise<string>;
  getSelfTempToken(): Promise<IGetTempTokenVo>;
  getSelectionRecords(selection, options?): Promise<IGetSelectionRecordsVo>;
}
```

**Event Sync (Host → Plugin):**
```typescript
interface IChildBridgeMethods {
  syncUIConfig(uiConfig: IUIConfig): void;
  syncSelection(selection?: ISelection): void;
  syncBasePermissions(permissions: IBasePermissions): void;
  syncUrlParams(urlParams: IUrlParams): void;
}
```

### 7.5 Plugin Authentication

```typescript
// Token request
POST /api/plugin/{pluginId}/token
Body: { secret, scopes, baseId }

Response: {
  accessToken: string,    // 10 minute expiry
  refreshToken: string,   // 30 day expiry
  scopes: string[]
}
```

### 7.6 Official Plugins

| Plugin | Positions | Purpose |
|--------|-----------|---------|
| **Chart** | dashboard, panel | Data visualization |
| **Sheet Form** | view | Spreadsheet-based forms |

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| Field Factory | `apps/nestjs-backend/src/features/field/model/factory.ts` |
| Link Service | `apps/nestjs-backend/src/features/calculation/link.service.ts` |
| Reference Service | `apps/nestjs-backend/src/features/calculation/reference.service.ts` |
| View Service | `apps/nestjs-backend/src/features/view/view.service.ts` |
| Filter Query | `apps/nestjs-backend/src/db-provider/filter-query/` |
| Plugin Service | `apps/nestjs-backend/src/features/plugin/plugin.service.ts` |
| Plugin Bridge | `packages/sdk/src/plugin-bridge/` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Database Architecture](./ARCHITECTURE-DATABASE.md) | [Service Layer](./ARCHITECTURE-SERVICES.md)*
