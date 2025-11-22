# Teable Service Layer Architecture

> **Detailed documentation of Domain-Driven Design patterns, service organization, and data migration.**

## Table of Contents

1. [Domain-Driven Design](#1-domain-driven-design)
2. [Service Organization](#2-service-organization)
3. [Domain Events](#3-domain-events)
4. [Data Migration & Schema Evolution](#4-data-migration--schema-evolution)
5. [Performance & Scalability](#5-performance--scalability)

---

## 1. Domain-Driven Design

### Overview

Teable's backend architecture is influenced by **Domain-Driven Design (DDD)** principles, though it takes a pragmatic rather than purist approach. DDD is particularly well-suited for Teable because:

- **Complex domain logic**: Spreadsheet operations involve intricate rules around field types, relationships, and calculations
- **Evolving requirements**: No-code platforms constantly add new field types, view types, and integrations
- **Team scalability**: Feature modules can be developed and maintained independently

The key insight is that Teable's domain is not "data management" generically—it's specifically "no-code database operations." This distinction shapes how entities, aggregates, and bounded contexts are defined.

### 1.1 Bounded Contexts

In DDD, a **bounded context** is a logical boundary within which a particular domain model applies. Teable implements bounded contexts as NestJS feature modules, each with its own services, controllers, and DTOs.

Teable organizes code into **45+ bounded contexts** (feature modules). Each context:
- Owns its domain logic and data models
- Exposes services that other contexts can import
- Has clear interfaces with other contexts (no internal implementation leakage)
- Can be reasoned about independently

```
features/
├── table/              # Table aggregate domain
├── record/             # Record aggregate domain
├── field/              # Field aggregate domain
├── view/               # View aggregate domain
├── base/               # Base/workspace domain
├── space/              # Space domain
├── user/               # User management domain
├── auth/               # Authentication domain
├── collaborator/       # Collaboration domain
├── calculation/        # Field calculation domain
├── graph/              # Dependency graph domain
├── aggregation/        # Data aggregation domain
├── import/             # Data import domain
├── export/             # Data export domain
├── attachment/         # File management domain
├── notification/       # Notification domain
├── plugin/             # Plugin system domain
├── dashboard/          # Dashboard domain
├── trash/              # Trash/recovery domain
├── comment/            # Record comments domain
├── share/              # Sharing domain
├── invitation/         # Invitation domain
├── oauth/              # OAuth domain
├── ai/                 # AI features domain
└── ... (20+ more)
```

### 1.2 Primary Aggregates

An **aggregate** in DDD is a cluster of domain objects that are treated as a single unit for data changes. The aggregate has a "root" entity that controls access to its members. In Teable:

- **Table is an aggregate** because fields and views belong to tables and must maintain consistency with the table
- **Record is an aggregate** because cell values must be valid for their field types
- **Field is an aggregate** because field options and references form a consistent unit

**Why aggregates matter**: When you delete a table, all its fields and views must be deleted. When you create a field, the table's version must increment. These transactional boundaries are enforced by making Table an aggregate root.

**Table Aggregate:**
```typescript
// Root: TableMeta
// Contains: Fields, Views
// Invariants: One primary field, unique field names within table

class TableService {
  async createTable(baseId: string, tableRo: ICreateTableRo) {
    // Enforce aggregate invariants
    // Create physical table in database
    // Create default primary field
    // Initialize default grid view
    // Emit TABLE_CREATE event for listeners
  }
}
```

The Table aggregate enforces that every table has exactly one primary field, field names are unique within the table, and views always have valid column configurations.

**Record Aggregate:**
```typescript
// Root: Record (physical row)
// Contains: Cell values, computed values
// Invariants: Valid field types, link integrity

class RecordService {
  async createRecords(tableId: string, records: IRecord[]) {
    // Validate field types
    // Resolve link references
    // Calculate computed fields
    // Emit RECORD_CREATE events
  }
}
```

**Field Aggregate:**
```typescript
// Root: Field
// Contains: Options, metadata, references
// Invariants: Valid type configuration, no circular refs

class FieldService {
  async createField(tableId: string, fieldRo: IFieldRo) {
    // Validate field configuration
    // Create physical column
    // Update reference graph
    // Emit FIELD_CREATE event
  }
}
```

### 1.3 Entity Definitions

Entities in DDD have identity—two entities with the same attributes are different if they have different IDs. Teable's entities include:

- **TableMeta**: Identity is `id`, has lifecycle operations
- **Field**: Identity is `id`, has type-specific behavior
- **View**: Identity is `id`, has type-specific rendering logic
- **Record**: Identity is `__id`, lives in physical tables

**Factory Pattern for Fields:**

The Field entity is polymorphic—a NumberField behaves differently from a LinkField. Teable uses the Factory pattern to instantiate the correct field class based on type. This encapsulates the type-switching logic and ensures each field has the right behavior.

```typescript
// Location: features/field/model/factory.ts
export function createFieldInstanceByVo(field: IFieldVo): IFieldInstance {
  switch (field.type) {
    case FieldType.SingleLineText:
      return plainToInstance(SingleLineTextFieldDto, field);
    case FieldType.Number:
      return plainToInstance(NumberFieldDto, field);
    case FieldType.Link:
      return plainToInstance(LinkFieldDto, field);
    case FieldType.Formula:
      return plainToInstance(FormulaFieldDto, field);
    // ... 16+ more types
  }
}
```

The factory uses `class-transformer`'s `plainToInstance` to convert plain objects (from database or API) into typed class instances with methods.

**Field Entity Structure:**
```typescript
class SingleLineTextFieldDto extends SingleLineTextFieldCore implements FieldBase {
  // Domain behavior
  get isStructuredCellValue() { return false; }

  // Value conversion
  convertCellValue2DBValue(value: unknown): unknown { ... }
  convertDBValue2CellValue(value: unknown): unknown { ... }

  // Validation
  validateCellValue(value: unknown): boolean { ... }
}
```

### 1.4 Value Objects

**Cell Values:**
```typescript
// Immutable value objects for cell data
interface ITextCellValue { type: 'text'; value: string; }
interface INumberCellValue { type: 'number'; value: number; }
interface ILinkCellValue { type: 'link'; value: { id: string; title: string; }[]; }
interface IAttachmentCellValue { type: 'attachment'; value: IAttachment[]; }
```

**Filter Objects:**
```typescript
interface IFilter {
  conjunction: 'and' | 'or';
  filterSet: (IFilterItem | IFilter)[];
}

interface IFilterItem {
  fieldId: string;
  operator: FilterOperator;
  value: FilterValue;
}
```

### 1.5 Table Domain Model

```typescript
// Location: features/table-domain/table-domain-query.service.ts
class TableDomain {
  id: string;
  fields: FieldCore[];

  // Domain methods
  getField(fieldId: string): FieldCore | undefined;
  getFieldByDbName(dbFieldName: string): FieldCore | undefined;
  getPrimaryField(): FieldCore;

  // Relationship resolution
  getAllRelatedTables(): Promise<Map<string, TableDomain>>;
}

// Construction
async getTableDomainById(tableId: string): Promise<TableDomain> {
  const tableMeta = await this.getTableMetaById(tableId);
  const fieldRaws = await this.getTableFields(tableMeta.id);
  return this.buildTableDomain(tableMeta, fieldRaws);
}
```

---

## 2. Service Organization

### Overview

Teable's service layer follows a **layered architecture** that separates concerns and enables testability. The key principle is **unidirectional dependency flow**: higher layers depend on lower layers, never the reverse.

This layering provides several benefits:
- **Testability**: Domain services can be unit tested without HTTP or database concerns
- **Flexibility**: Infrastructure can be swapped (e.g., PostgreSQL to SQLite) without changing business logic
- **Clarity**: Each layer has a clear responsibility, making code navigation easier

### 2.1 Layered Architecture

Each layer has distinct responsibilities and communicates only with adjacent layers:

```
┌──────────────────────────────────────────────────────────────┐
│                    API Layer (Controllers)                    │
│  • HTTP request handling and routing                          │
│  • Request validation (Zod schemas)                           │
│  • Response formatting and error mapping                      │
│  • Authentication/authorization guards                        │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│               Application Services (OpenAPI Services)         │
│  • Use case orchestration (coordinate multiple domain calls)  │
│  • Transaction boundaries (what succeeds/fails together)      │
│  • Cross-domain coordination (e.g., create table + fields)    │
│  • Event emission after successful operations                 │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                   Domain Services                             │
│  • Core business logic (validation, calculations)             │
│  • Aggregate operations (CRUD on domain entities)             │
│  • Domain invariant enforcement (business rules)              │
│  • No knowledge of HTTP or external systems                   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│               Infrastructure Services                         │
│  • Database access (Prisma, Knex)                             │
│  • Cache management (Redis, in-memory)                        │
│  • External integrations (S3, email, OAuth providers)         │
│  • Low-level operations that domain shouldn't know about      │
└──────────────────────────────────────────────────────────────┘
```

**Why "OpenAPI Services"?** Teable distinguishes between internal domain services and external-facing "OpenAPI" services. The OpenAPI services are specifically designed for the REST API contract and handle concerns like field key translation, pagination, and response shaping.

### 2.2 Service Types

**Application Services (Orchestration):**
```typescript
// Location: features/record/record-modify/record-modify.service.ts
@Injectable()
export class RecordModifyService {
  constructor(
    private readonly createService: RecordCreateService,
    private readonly updateService: RecordUpdateService,
    private readonly deleteService: RecordDeleteService,
  ) {}

  async updateRecords(tableId: string, updateRecordsRo: IUpdateRecordsRo) {
    // 1. Validate input
    // 2. Coordinate domain services
    // 3. Handle transactions
    // 4. Emit events
  }
}
```

**Domain Services (Business Logic):**
```typescript
// Location: features/calculation/link.service.ts (59KB!)
@Injectable()
export class LinkService {
  // Core domain logic for relationships
  async createLinkField(tableId: string, field: ILinkFieldOptions) { ... }
  async updateLinkCellValues(tableId: string, recordId: string, values: ILinkCellValue[]) { ... }
  async resolveLinkReferences(tableId: string, linkFieldId: string) { ... }
}
```

**Shared Services (Cross-Cutting):**
```typescript
// Location: features/record/record-modify/record-modify-shared.service.ts
@Injectable()
export class RecordModifySharedService {
  validateFieldsAndTypecast(tableId: string, records: IRecord[]): Promise<IRecord[]>;
  appendRecordOrderIndexes(tableId: string, records: IRecord[]): Promise<IRecord[]>;
  generateCellContexts(changes: IChange[]): ICellContext[];
  formatChangesToOps(tableId: string, changes: IChange[]): IRawOp[];
}
```

### 2.3 Module Structure

**Feature Module Pattern:**
```typescript
// Location: features/table/table.module.ts
@Module({
  imports: [
    CalculationModule,
    FieldModule,
    RecordModule,
    ViewModule,
  ],
  providers: [
    TableService,
    DbProvider,
    TablePermissionService,
  ],
  exports: [
    FieldModule,
    RecordModule,
    ViewModule,
    TableService,
    TablePermissionService,
  ],
})
export class TableModule {}
```

**Open API Module Pattern:**
```typescript
// Location: features/record/open-api/record-open-api.module.ts
@Module({
  imports: [RecordModule, ShareDbModule],
  controllers: [RecordOpenApiController],
  providers: [RecordOpenApiService],
  exports: [RecordOpenApiService],
})
export class RecordOpenApiModule {}
```

### 2.4 Dependency Injection

**Service Composition:**
```typescript
@Injectable()
export class RecordCreateService {
  constructor(
    // Infrastructure
    private readonly prismaService: PrismaService,
    private readonly cls: ClsService<IClsStore>,

    // Domain services
    private readonly recordService: RecordService,
    private readonly linkService: LinkService,
    private readonly batchService: BatchService,

    // Shared services
    private readonly shared: RecordModifySharedService,

    // Orchestration
    private readonly computedOrchestrator: ComputedOrchestratorService,
  ) {}
}
```

### 2.5 Statistics

| Metric | Count |
|--------|-------|
| Feature Modules | 83 |
| Service Classes | 110+ |
| Controllers | 45+ |
| Domain Events | 30+ |

---

## 3. Domain Events

### Overview

Domain events are notifications that something significant happened in the domain. In Teable, events serve multiple purposes:

1. **Decoupling**: Modules can react to events without direct dependencies (e.g., history tracking doesn't need to be called explicitly)
2. **Audit trail**: Events provide a record of what changed and when
3. **Real-time sync**: Events trigger WebSocket broadcasts to connected clients
4. **Automation**: User-defined automations subscribe to events
5. **Derived data**: Computed fields and lookups recalculate based on events

The event system follows the **Observer pattern**: services emit events, and listeners subscribe to events they care about. This enables loose coupling—the record service doesn't know (or care) that the history service is tracking changes.

### 3.1 Event System Architecture

Teable uses NestJS's built-in event emitter with custom event classes. Events are strongly typed, providing compile-time safety for event payloads.

```typescript
// Location: event-emitter/events/event.enum.ts
enum Events {
  // Table lifecycle
  TABLE_CREATE = 'table.create',
  TABLE_UPDATE = 'table.update',
  TABLE_DELETE = 'table.delete',

  // Field lifecycle
  TABLE_FIELD_CREATE = 'table.field.create',
  TABLE_FIELD_UPDATE = 'table.field.update',
  TABLE_FIELD_DELETE = 'table.field.delete',

  // Record lifecycle
  TABLE_RECORD_CREATE = 'table.record.create',
  TABLE_RECORD_UPDATE = 'table.record.update',
  TABLE_RECORD_DELETE = 'table.record.delete',

  // View lifecycle
  TABLE_VIEW_CREATE = 'table.view.create',
  TABLE_VIEW_UPDATE = 'table.view.update',
  TABLE_VIEW_DELETE = 'table.view.delete',

  // User lifecycle
  USER_SIGNIN = 'user.signin',
  USER_SIGNUP = 'user.signup',
  USER_SIGNOUT = 'user.signout',

  // Collaboration
  COLLABORATOR_CREATE = 'collaborator.create',
  COLLABORATOR_DELETE = 'collaborator.delete',
}
```

### 3.2 Event Structure

```typescript
// Base event class
abstract class CoreEvent<Payload extends object> {
  abstract name: Events;
  payload: Payload;
  context: IEventContext;
  isBulk: boolean;
  id: string;
}

// Operation events
abstract class OpEvent<Payload extends object> extends CoreEvent<Payload> {
  abstract rawOpType: RawOpType;  // Create, Edit, Delete
}

// Concrete event
class RecordCreateEvent extends OpEvent<IRecordCreatePayload> {
  name = Events.TABLE_RECORD_CREATE;
  rawOpType = RawOpType.Create;
}
```

### 3.3 Event Publishing

```typescript
// Location: event-emitter/event-emitter.service.ts
@Injectable()
export class EventEmitterService {
  // Convert ShareDB operations to domain events
  async ops2Event(rawOpMaps?: IRawOpMap[]): Promise<void> {
    const events = this.collectEventsFromRawOpMap(rawOpMaps);
    const grouped = this.groupEventsByType(events);
    const aggregated = this.aggregateRelatedEvents(grouped);

    for (const event of aggregated) {
      this.eventEmitter.emit(event.name, event);
    }
  }
}
```

### 3.4 Event Listeners

```typescript
// Location: event-emitter/listeners/

// Record history tracking
@Injectable()
export class RecordHistoryListener {
  @OnEvent(Events.TABLE_RECORD_UPDATE, { async: true })
  async handleRecordUpdate(event: RecordUpdateEvent) {
    await this.createHistoryEntry(event);
  }
}

// Notification sending
@Injectable()
export class CollaboratorNotificationListener {
  @OnEvent(Events.TABLE_RECORD_CREATE, { async: true })
  async handleRecordCreate(event: RecordCreateEvent) {
    await this.notifySubscribers(event);
  }
}

// Automation triggers
@Injectable()
export class ActionTriggerListener {
  @OnEvent(Events.TABLE_RECORD_UPDATE, { async: true })
  async handleRecordUpdate(event: RecordUpdateEvent) {
    await this.executeAutomations(event);
  }
}

// Trash management
@Injectable()
export class TrashListener {
  @OnEvent(Events.TABLE_DELETE, { async: true })
  async handleTableDelete(event: TableDeleteEvent) {
    await this.moveToTrash(event);
  }
}
```

### 3.5 Event Aggregation

```typescript
// Combine multiple events into bulk events
private combineEvents(events: OpEvent[]): OpEvent {
  if (events.length <= 1) return events[0];

  // Merge payloads
  return events.reduce((combined, event) => {
    const changes = this.mergeChanges(combined, event);
    combined.payload = { ...combined.payload, ...changes };
    combined.isBulk = true;
    return combined;
  });
}
```

---

## 4. Data Migration & Schema Evolution

### Overview

Data migration is critical for any evolving application. Teable needs to:
- Add new features without breaking existing data
- Support both SQLite (development) and PostgreSQL (production)
- Enable zero-downtime deployments
- Maintain data integrity across schema changes

Teable uses **Prisma Migrate** for schema migrations, with a custom template system that generates database-specific migrations. This approach enables:
- **Single source of truth**: One template schema defines both SQLite and PostgreSQL schemas
- **Database-specific optimizations**: PostgreSQL migrations can use features SQLite lacks
- **Versioned history**: Every schema change is tracked and reversible

### 4.1 Migration Workflow

The migration process involves multiple steps because Teable supports two databases. Here's the typical workflow:

**Prisma Migration Process:**
```bash
# 1. Modify the template schema (the source of truth)
edit packages/db-main-prisma/prisma/template.prisma

# 2. Generate database-specific schemas from template
make gen-prisma-schema
# This produces: prisma/postgres/schema.prisma and prisma/sqlite/schema.prisma

# 3. Create migration files (generates SQL)
make db-migration
# This creates timestamped migration directories with SQL files

# 4. Apply migrations to your database
make sqlite.mode     # Development (SQLite)
make postgres.mode   # Production (PostgreSQL)
```

**Why a template schema?** SQLite and PostgreSQL have different syntax and capabilities. The template uses a common subset, and the generator handles database-specific translations (e.g., `JSONB` becomes `TEXT` in SQLite).

### 4.2 Migration History

Teable's migration history tells the story of its evolution. Each migration is a snapshot of a feature addition, bug fix, or performance improvement.

**56+ Migrations** covering:
- **Initial schema setup**: Core tables (Space, Base, TableMeta, Field, View, etc.)
- **Feature additions**: Conditional lookups, AI field config, button fields
- **Performance indexes**: Optimizing common query patterns
- **Constraint modifications**: Adding unique constraints, foreign keys
- **Data repairs**: Fixing historical data issues from bugs

**Migration Naming Convention:**
```
YYYYMMDDHHmmss_<description>
20250922120000_add_conditional_lookup_flag    # Feature flag for new lookup behavior
20250905035737_add_trash_index                # Performance: index for trash queries
20250828083308_add_app_robot_user             # Feature: system users for automation
```

**Reading migration history** is valuable for understanding:
- When a feature was introduced (to understand backwards compatibility)
- What indexes exist (for query optimization)
- How data structures evolved (for debugging data issues)

### 4.3 Zero-Downtime Strategies

Production deployments require migrations that don't lock tables or cause errors for running applications. Teable uses several patterns to achieve zero-downtime migrations:

**Pattern 1: Add Nullable Column, Then Backfill**

This is the safest approach for adding new columns:

```sql
-- Step 1: Add nullable column (fast, no table lock)
ALTER TABLE "field" ADD COLUMN "is_conditional_lookup" BOOLEAN;

-- Step 2: Backfill data in batches (can run while app is live)
UPDATE "field" SET is_conditional_lookup = false WHERE is_lookup = true;

-- Step 3: Add constraints later (separate deployment if needed)
-- ALTER TABLE "field" ALTER COLUMN "is_conditional_lookup" SET NOT NULL;
```

**Why nullable first?** Adding a NOT NULL column would require a default value and could lock large tables. Nullable columns can be added instantly, then filled asynchronously.

**Pattern 2: Safe Unique Constraint Addition**

Adding unique constraints can fail if duplicates exist. The pattern handles this:

```sql
BEGIN;
-- First, identify and remove/merge duplicates
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER (PARTITION BY unique_cols ORDER BY created_time) as rn
  FROM table
)
DELETE FROM table WHERE id IN (SELECT id FROM duplicates WHERE rn > 1);

-- Then safely add the constraint
CREATE UNIQUE INDEX "idx_unique" ON "table"(unique_cols);
COMMIT;
```

**Pattern 3: Concurrent Index Creation (PostgreSQL)**

For large tables, indexes can be created without blocking writes:
```sql
CREATE INDEX CONCURRENTLY "idx_name" ON "table"(column);
```

### 4.4 Deployment Integration

**Docker Startup Script:**
```bash
# scripts/start.sh
case "$1" in
  "skip-migrate")
    # Skip if pre-applied
    ;;
  "migrate-only")
    run_migration && exit 0
    ;;
  *)
    run_migration  # Default: migrate then start
    ;;
esac
```

**Migration Script (db-migrate.mjs):**
- Validates PostgreSQL connection
- Retries 5 times with 3s delays
- Executes `prisma migrate deploy`

### 4.5 Schema Versioning

```prisma
// Entity-level versioning
model TableMeta {
  version Int @default(1)
}

model Field {
  version Int @default(1)
}

// Operation-level versioning (for OT)
model Ops {
  version Int
  @@unique([collection, docId, version])
}
```

### 4.6 Backward Compatibility

**JSON Metadata Strategy:**
- Extensible field options without migrations
- Feature flags via boolean columns
- Nullable columns for optional features

**API Compatibility:**
- No breaking changes to existing endpoints
- New features as new endpoints
- Deprecation via documentation

---

## 5. Performance & Scalability

### 5.1 Database Optimization

**Query Optimization:**
```typescript
// Batch loading with DataLoader pattern
@Injectable()
export class DataLoaderService {
  async batchLoadRecords(ids: string[]): Promise<IRecord[]> {
    return this.prisma.record.findMany({
      where: { id: { in: ids } }
    });
  }
}
```

**Connection Pooling:**
```typescript
// Prisma configuration
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // Pool managed by Prisma
}
```

### 5.2 Batch Processing

```typescript
// Chunk large operations
const chunkSize = this.thresholdConfig.calcChunkSize;
for (let i = 0; i < records.length; i += chunkSize) {
  const chunk = records.slice(i, i + chunkSize);
  await this.processChunk(chunk);
}
```

### 5.3 Computed Field Orchestration

```typescript
// Location: features/record/computed/services/computed-orchestrator.service.ts
async computeCellChangesForRecords(tableId, cellContexts, update) {
  // 1. Collect affected computed fields
  const impact = await this.collector.collect(tableId, cellContexts);

  // 2. Execute update in transaction
  await update();

  // 3. Recalculate affected fields
  await this.evaluator.evaluate(impact);
}
```

### 5.4 Horizontal Scaling

**Stateless Design:**
- Sessions externalized to Redis
- No server-side state
- Load balancer compatible

**Multi-Instance Support:**
- Redis pub/sub for real-time sync
- Shared cache layer
- Database connection pooling

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| Event Emitter | `apps/nestjs-backend/src/event-emitter/` |
| Event Listeners | `apps/nestjs-backend/src/event-emitter/listeners/` |
| Domain Models | `packages/core/src/models/` |
| Field Factory | `apps/nestjs-backend/src/features/field/model/factory.ts` |
| Record Services | `apps/nestjs-backend/src/features/record/` |
| Table Domain | `apps/nestjs-backend/src/features/table-domain/` |
| Prisma Schema | `packages/db-main-prisma/prisma/template.prisma` |
| Migrations | `packages/db-main-prisma/prisma/postgres/migrations/` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Core Architecture](./ARCHITECTURE-CORE.md) | [Database Architecture](./ARCHITECTURE-DATABASE.md)*
