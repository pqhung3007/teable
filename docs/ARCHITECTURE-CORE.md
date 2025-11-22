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

### Overview

Teable's data persistence layer solves a fundamental challenge: **how to provide spreadsheet-like flexibility while maintaining database-level performance and reliability**. Traditional databases require fixed schemas defined upfront, but no-code platforms need users to add, remove, and modify columns on the fly without writing SQL.

The solution is a **two-tier architecture** that separates schema metadata from actual data storage. This allows Teable to:
- Let users create and modify fields instantly without downtime
- Leverage native database features (indexes, constraints, generated columns) for performance
- Support 20+ field types with type-specific storage and querying
- Enable real-time collaboration through optimistic locking and versioning

### 1.1 Dynamic Schema Management

Teable uses a **hybrid approach** combining metadata tables with dynamically created physical tables. The key insight is that while users see "tables" and "fields," the system actually maintains two parallel representations:

1. **Metadata Layer**: Prisma-managed tables (`TableMeta`, `Field`, `View`) that store configuration, types, and relationships. These are schema-stable and define what data looks like.

2. **Physical Layer**: Dynamically created database tables (e.g., `tbl_abc123`) with columns generated at runtime. These hold the actual user data and leverage native database features.

This separation provides several benefits:
- **Schema changes are fast**: Adding a field just requires an `ALTER TABLE` rather than migrating a fixed schema
- **Type safety is preserved**: Each field type maps to an appropriate database type (VARCHAR, NUMERIC, JSONB, etc.)
- **Indexing is native**: Physical columns can be indexed for query performance
- **Database features work**: Foreign keys, constraints, and generated columns all function normally

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

The field type system is the heart of Teable's flexibility. Each field type encapsulates:
- **Storage strategy**: How data is physically stored in the database
- **Value conversion**: How to transform between user-facing values and database values
- **Validation rules**: What constitutes valid input for this type
- **Query behavior**: How filtering, sorting, and aggregation work for this type

The design follows a **Strategy Pattern** where each field type implements a common interface but provides type-specific behavior. This allows the system to handle a `Number` field completely differently from a `Link` field while presenting a unified API.

**20+ Field Types Supported:**

| Category | Field Types | Storage Type | Rationale |
|----------|-------------|--------------|-----------|
| **Text** | SingleLineText, LongText | VARCHAR, TEXT | Native string operations, full-text search |
| **Numeric** | Number, Rating, AutoNumber | NUMERIC, INTEGER, SERIAL | Precise arithmetic, native aggregations |
| **Selection** | SingleSelect, MultipleSelect | VARCHAR, JSONB | Single values as string; multiple as JSON array |
| **Date/Time** | Date, CreatedTime, LastModifiedTime | TIMESTAMP | Timezone-aware, supports date math |
| **Boolean** | Checkbox | BOOLEAN | Native boolean operations |
| **Files** | Attachment | JSONB | Flexible metadata structure per file |
| **Relations** | Link, User | JSONB | Store reference IDs with denormalized titles |
| **Computed** | Formula, Lookup, Rollup | GENERATED or VIEW | See formula engine for evaluation strategy |
| **System** | CreatedBy, LastModifiedBy | VARCHAR | Automatic audit trail |
| **Interactive** | Button | N/A | UI-only, triggers automations |

**Why JSONB for complex types?** PostgreSQL's JSONB provides the best of both worlds: flexible document storage with indexable, queryable access. For multi-value fields like attachments or links, this avoids the need for separate junction tables while still supporting operators like `@>` (contains) for filtering.

### 1.3 Field Metadata Storage

Field configuration is stored as JSON in the `options` column, enabling extensibility without schema migrations. Each field type defines its own options structure, and the backend validates these during field creation/update. This pattern allows:

- Adding new field options without database migrations
- Type-specific configuration (e.g., `precision` for numbers, `choices` for selects)
- Forward compatibility when new features are added

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

Version tracking is critical for a collaborative platform where multiple users may be editing the same data simultaneously. Teable implements versioning at three distinct levels, each serving a different purpose:

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

**Why three version levels?**

| Level | Purpose | Incremented When |
|-------|---------|------------------|
| **TableMeta.version** | Optimistic locking for schema changes | Field added/removed, table renamed |
| **Field.version** | Track field configuration changes | Field options modified |
| **Ops.version** | Operational Transformation ordering | Any data operation via ShareDB |

**How versioning enables collaboration:**
1. **Conflict detection**: Before applying a change, check if the version matches. If not, another user modified the resource.
2. **Cache invalidation**: When a version changes, cached data for that resource becomes stale.
3. **Real-time sync**: ShareDB uses operation versions to order and transform concurrent edits.

---

## 2. Record Storage

### Overview

Record storage in Teable balances flexibility with performance. Unlike traditional ORMs where each model has a fixed structure, Teable's records are stored in dynamically-created tables where columns are added at runtime. This section explains how individual rows are structured and how the system maintains data integrity.

### 2.1 Row-Level Data Structure

Every physical table includes a set of **system columns** that Teable manages automatically, plus **user-defined columns** created by field definitions. The system columns provide:

- **Identity**: A unique record ID (`__id`) that persists even if the record is moved or renamed
- **Ordering**: An auto-increment number (`__auto_number`) for stable default sorting
- **Audit trail**: Timestamps and user IDs for creation and modification
- **Concurrency control**: A version number (`__version`) for optimistic locking

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

The storage strategy for each field type is carefully chosen to balance query performance with flexibility. The key principle is: **use the most specific database type that can represent the data**. This enables native database operations (sorting numbers numerically, comparing dates chronologically) while preserving the flexibility needed for complex types.

| Field Type | Storage | Example Value | Why This Type |
|------------|---------|---------------|---------------|
| SingleLineText | VARCHAR | `'Hello World'` | Supports LIKE queries, indexing |
| Number | NUMERIC | `123.45` | Precise arithmetic, native comparisons |
| Checkbox | BOOLEAN | `true` | Native boolean logic |
| SingleSelect | VARCHAR | `'Option A'` | Fast equality checks, low storage |
| MultipleSelect | JSONB | `['Option A', 'Option B']` | Array containment queries |
| Date | TIMESTAMP | `'2024-01-15T10:30:00Z'` | Date arithmetic, timezone support |
| Attachment | JSONB | `[{id, name, size, token, path}]` | Flexible metadata per file |
| Link | JSONB | `[{id: 'rec_xxx', title: '...'}]` | Denormalized for display performance |
| User | JSONB | `[{id: 'usr_xxx', title: '...'}]` | Consistent with link structure |

**Denormalization in Link/User fields**: Notice that link and user fields store both the `id` and a `title`. This is intentional denormalization—it avoids JOIN queries when displaying linked records in the UI. The title is updated asynchronously when the source record changes.

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

### Overview

Relationships are one of the most complex aspects of a no-code database. Unlike traditional databases where relationships are defined by foreign keys at design time, Teable allows users to create links between any tables at runtime. The system must handle:

- **Bidirectional links**: When Table A links to Table B, Table B should automatically show which records in Table A reference it
- **Many-to-many relationships**: Users can link multiple records without understanding junction tables
- **Cascading updates**: When a linked record's title changes, all references should update
- **Referential integrity**: Deleted records must be unlinked from all referring records

The design philosophy is to **hide database complexity while preserving database power**. Users see a simple "link" field; behind the scenes, Teable manages foreign keys, junction tables, and cascade operations.

### 3.1 Link Fields Between Tables

Every link field creates a **bidirectional relationship** by default. When you create a link from Table A to Table B, Teable automatically creates a corresponding "symmetric" field in Table B pointing back to Table A. This ensures users can navigate relationships from either direction.

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

The relationship type determines how data is physically stored:

**One-to-One / One-to-Many**: The "many" side stores the foreign key directly in a JSONB column. No additional tables are needed.

**Many-to-Many**: Requires a junction table to store the relationship pairs. Teable creates and manages this automatically:

```sql
-- Auto-created for manyToMany relationships
CREATE TABLE "junction_tblA_tblB" (
  "fld_linkA" VARCHAR REFERENCES "tbl_A"("__id"),
  "fld_linkB" VARCHAR REFERENCES "tbl_B"("__id"),
  PRIMARY KEY ("fld_linkA", "fld_linkB")
);
```

**Why junction tables for many-to-many?** While JSONB arrays could store multiple references, junction tables enable:
- Database-enforced referential integrity via foreign keys
- Efficient queries for "find all records linked to X"
- Standard indexing on both sides of the relationship

**LinkService Responsibilities:**
The `LinkService` (at 59KB, one of the largest services) handles all relationship complexity:
- Create symmetric link fields in both tables
- Maintain junction tables (create, populate, drop)
- Cascade link value updates when titles change
- Handle link field deletion (clean up symmetric field and junction table)
- Resolve self-referential links (a table linking to itself)

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

### Overview

Views in Teable are **saved query configurations** that determine how data is displayed and filtered. Unlike materialized views in traditional databases, Teable views are evaluated at query time—they don't duplicate data but rather store the parameters (filters, sorts, field visibility) that shape the query.

This design enables:
- **Multiple perspectives on the same data**: Sales team sees one view, Operations sees another, but changes sync instantly
- **No data duplication**: Views are cheap to create and don't consume storage
- **Real-time consistency**: Data updates appear in all views immediately

Each view type provides a different visualization paradigm while sharing the same underlying filtering and sorting infrastructure.

### 4.1 View Types

| Type | Purpose | Key Options | Backend Considerations |
|------|---------|-------------|------------------------|
| **Grid** | Spreadsheet-like | `rowHeight`, `frozenColumnCount` | Supports all filter/sort/group operations |
| **Kanban** | Card-based boards | `stackFieldId`, `coverFieldId` | Groups by single-select field, requires special aggregation |
| **Gallery** | Visual grid | `coverFieldId`, `isCoverFit` | Optimized for attachment field display |
| **Calendar** | Date-based | `startDateFieldId`, `endDateFieldId` | Date range queries, recurring event support |
| **Form** | Data entry | `coverUrl`, `logoUrl`, `submitLabel` | Write-only view, public sharing support |
| **Plugin** | Custom views | `pluginId`, `pluginInstallId` | Delegated to plugin system |

**Note on Form views**: Unlike other views, forms are primarily for data input rather than display. They can be shared publicly without authentication and have their own permission model.

### 4.2 Server-Side Filtering

Filtering is one of the most complex features in the query engine. The filter system must:
- Support nested boolean logic (AND/OR groups within groups)
- Handle 40+ operators across different field types
- Generate efficient SQL for both PostgreSQL and SQLite
- Support dynamic field references (filter by "another field's value")

The filter structure is recursive, allowing arbitrarily nested conditions:

**Filter Structure:**
```typescript
interface IFilter {
  conjunction: 'and' | 'or';
  filterSet: (IFilterItem | IFilter)[];  // Nested groups allow complex logic
}

interface IFilterItem {
  fieldId: string;
  operator: FilterOperator;
  value: any;
  isSymbol?: boolean;  // When true, value is a fieldId to compare against
}
```

**Why `isSymbol`?** This enables filters like "Show records where Due Date is after Created Date"—comparing two fields rather than a field to a constant value.

**40+ Filter Operators by Category:**

| Category | Operators | Notes |
|----------|-----------|-------|
| Text | `is`, `isNot`, `contains`, `doesNotContain`, `startsWith`, `endsWith` | Case-insensitive by default |
| Numbers | `isGreater`, `isLess`, `isGreaterEqual`, `isLessEqual` | Handles NULL gracefully |
| Dates | `isBefore`, `isAfter`, `isWithIn`, `today`, `pastWeek`, etc. | Dynamic operators like `today` evaluated at query time |
| Multi-value | `isAnyOf`, `hasAllOf`, `isExactly` | Array intersection/containment logic |
| Null | `isEmpty`, `isNotEmpty` | Works across all field types |

**Filter Query Generation**: Each field type has a `FilterAdapter` that knows how to generate SQL for its operators. For example, filtering a JSONB multi-select field uses different SQL than filtering a VARCHAR text field.

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

### Overview

Teable exposes a RESTful API that follows resource-oriented design principles. The API serves three distinct audiences:

1. **Internal Frontend**: The Next.js web application makes API calls for all operations
2. **External Integrations**: Third-party applications can automate data operations
3. **Plugins**: Dashboard widgets and custom views access data through the API

The API design prioritizes:
- **Consistency**: Same patterns across all resource types
- **Discoverability**: Hierarchical URLs reflect data relationships
- **Efficiency**: Bulk operations reduce HTTP round-trips
- **Security**: Fine-grained permissions checked at every endpoint

### 5.1 REST API Design

The URL structure follows a **hierarchical pattern** that mirrors the data model. Each level of nesting represents a containment relationship:

**Resource Naming:**
```
/api/space                     # Workspaces (top-level containers)
/api/base                      # Databases within spaces
/api/table/{tableId}           # Tables within bases
/api/table/{tableId}/field     # Fields (columns) within tables
/api/table/{tableId}/view      # Views within tables
/api/table/{tableId}/record    # Records (rows) within tables
```

**Why table-centric URLs?** Most operations are table-scoped. By making `tableId` part of the URL, we:
- Simplify permission checking (table permissions cascade to fields/views/records)
- Enable efficient caching (invalidate by table)
- Provide clear context in logs and debugging

**CRUD Patterns:**
```typescript
GET    /api/table/{tableId}/record          // List records
POST   /api/table/{tableId}/record          // Create records
GET    /api/table/{tableId}/record/{id}     // Get single record
PATCH  /api/table/{tableId}/record/{id}     // Update record
DELETE /api/table/{tableId}/record/{id}     // Delete record
```

### 5.2 Bulk Operations

Spreadsheet-like applications often need to create, update, or delete hundreds of records at once. Making individual API calls for each record would be prohibitively slow. Teable's bulk operations address this with:

- **Transactional batching**: All records in a bulk operation succeed or fail together
- **Optimized SQL**: Single multi-row INSERT/UPDATE instead of individual statements
- **Event aggregation**: Domain events are combined to reduce downstream processing

```typescript
// Bulk create - up to 1000 records per request
POST /api/table/{tableId}/record
Body: { records: [{ fields: {...} }, ...], fieldKeyType?: 'id' | 'name' }

// Bulk update - atomic, all-or-nothing
PATCH /api/table/{tableId}/record
Body: { records: [{ id: '...', fields: {...} }, ...] }

// Bulk delete - triggers cascade cleanup
DELETE /api/table/{tableId}/record
Body: { recordIds: ['rec_xxx', 'rec_yyy'] }
```

**The `fieldKeyType` option**: By default, fields are identified by ID (`fld_xxx`). Setting `fieldKeyType: 'name'` allows using human-readable field names, which is more convenient for integrations but requires an extra lookup step.

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

### Overview

Teable's plugin system allows extending functionality without modifying core code. Plugins are **sandboxed web applications** that run in iframes and communicate with Teable through a bridge API. This architecture provides:

- **Security isolation**: Plugins can't access data they're not authorized for
- **Independent deployment**: Plugins can be updated without Teable releases
- **Developer flexibility**: Plugins can use any frontend framework
- **Graceful degradation**: A broken plugin doesn't crash the main application

The plugin model is inspired by VS Code extensions and Figma plugins—providing rich capabilities while maintaining security boundaries.

### 7.1 Plugin Architecture

The architecture follows a **host-guest model** where Teable (the host) embeds plugins (guests) in iframes and mediates all communication:

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

**Why iframes?** Iframes provide browser-native security isolation. A plugin cannot access Teable's DOM, cookies, or localStorage. All data access must go through the bridge API, which enforces permissions.

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
