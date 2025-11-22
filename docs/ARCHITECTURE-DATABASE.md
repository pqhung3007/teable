# Teable Database Architecture

> **Detailed documentation of database schema design, query engine, and formula system.**

## Table of Contents

1. [Database Schema Design](#1-database-schema-design)
2. [Dynamic Field Storage](#2-dynamic-field-storage)
3. [Query Engine](#3-query-engine)
4. [Formula Engine](#4-formula-engine)
5. [Performance Optimization](#5-performance-optimization)

---

## 1. Database Schema Design

### 1.1 Core Model Hierarchy

```
Space (workspace)
└── Base (database container)
    └── TableMeta (table definition)
        ├── Field (column definitions)
        ├── View (view configurations)
        └── Records (via physical table)
```

### 1.2 Key Models

#### Space (Workspace)
```prisma
model Space {
  id          String   @id @default(cuid())
  name        String
  credit      Float?
  deletedTime DateTime?
  createdTime DateTime @default(now())
  createdBy   String
  lastModifiedBy String?
}
```

#### Base (Database)
```prisma
model Base {
  id             String   @id @default(cuid())
  spaceId        String
  name           String
  order          Float
  schemaPass     String?       // Schema password
  icon           String?
  deletedTime    DateTime?
  createdTime    DateTime @default(now())
  createdBy      String
}
```

#### TableMeta (Table Definition)
```prisma
model TableMeta {
  id              String   @id @default(cuid())
  baseId          String
  name            String
  description     String?
  icon            String?
  dbTableName     String        // Physical table name in database
  version         Int      @default(1)
  order           Float
  deletedTime     DateTime?
  createdTime     DateTime @default(now())
  lastModifiedTime DateTime?
  createdBy       String
}
```

#### Field (Column Definition)
```prisma
model Field {
  id                    String   @id @default(cuid())
  tableId               String
  name                  String
  description           String?
  type                  String        // FieldType enum
  options               String?       // JSON field options
  isLookup              Boolean?
  lookupLinkedFieldId   String?
  lookupOptions         String?       // JSON lookup config
  aiConfig              String?       // AI configuration
  isConditionalLookup   Boolean?
  meta                  String?       // JSON metadata
  dbFieldName           String        // Physical column name
  dbViewName            String?       // View name for lookups
  isPrimary             Boolean?
  notNull               Boolean?
  unique                Boolean?
  isComputed            Boolean?
  hasError              Boolean?
  dbFieldType           String        // Database column type
  cellValueType         String        // CellValueType enum
  isMultipleCellValue   Boolean?
  version               Int      @default(1)
  order                 Float
  deletedTime           DateTime?
  createdTime           DateTime @default(now())
  lastModifiedTime      DateTime?
  createdBy             String
}
```

### 1.3 Supported Field Types

| Field Type | Storage | Cell Value Type | Description |
|------------|---------|-----------------|-------------|
| `SingleLineText` | VARCHAR | String | Single line text |
| `LongText` | TEXT | String | Multi-line text with rich formatting |
| `Number` | NUMERIC | Number | Numbers with formatting options |
| `SingleSelect` | VARCHAR | String | Single selection from options |
| `MultipleSelect` | JSONB | String[] | Multiple selections |
| `Date` | TIMESTAMP | DateTime | Date/time with timezone |
| `Checkbox` | BOOLEAN | Boolean | True/false values |
| `Rating` | INTEGER | Number | Star rating (1-10) |
| `Attachment` | JSONB | Object[] | File attachments |
| `Link` | JSONB | Object[] | Links to other tables |
| `Formula` | GENERATED/COMPUTED | Varies | Calculated fields |
| `Lookup` | VIEW | Varies | Values from linked records |
| `Rollup` | VIEW | Varies | Aggregated values from links |
| `User` | JSONB | Object[] | User references |
| `CreatedTime` | TIMESTAMP | DateTime | Auto-generated |
| `LastModifiedTime` | TIMESTAMP | DateTime | Auto-updated |
| `CreatedBy` | VARCHAR | String | Auto-generated |
| `LastModifiedBy` | VARCHAR | String | Auto-updated |
| `AutoNumber` | SERIAL | Number | Auto-increment |
| `Button` | N/A | N/A | Action trigger |

### 1.4 Multi-Tenancy Approach

Teable uses **workspace-based isolation**:

```
User → AccessToken/Session → Collaborator → Space/Base
```

**No explicit Organization model** - Instead:
- **Space** is the top-level container
- **Collaborator** provides fine-grained access at Space or Base level
- **Access Tokens** can be scoped to specific spaces/bases

---

## 2. Dynamic Field Storage

### 2.1 Physical Table Creation

When a logical table is created, Teable:

1. Creates a `TableMeta` record with a generated `dbTableName` (e.g., `tbl_xxx`)
2. Creates the physical table with system columns:

```sql
CREATE TABLE "tbl_xxx" (
  "__id" VARCHAR PRIMARY KEY,           -- Record ID
  "__auto_number" SERIAL,               -- Auto-increment
  "__created_time" TIMESTAMP,           -- Creation timestamp
  "__last_modified_time" TIMESTAMP,     -- Last modification
  "__created_by" VARCHAR,               -- Creator user ID
  "__last_modified_by" VARCHAR,         -- Last modifier user ID
  "__version" INTEGER DEFAULT 1         -- Optimistic locking
);
```

### 2.2 Dynamic Column Addition

When a field is added:

```typescript
// Field creation triggers DDL
const createColumnSchema = dbProvider.createColumnSchema(
  dbTableName,
  dbFieldName,      // e.g., "fld_abc123"
  dbFieldType,      // e.g., "VARCHAR", "JSONB"
  options           // NOT NULL, DEFAULT, etc.
);

// Executed as raw SQL
await prisma.$executeRawUnsafe(createColumnSchema);
```

### 2.3 Generated Columns for Formulas

Simple formulas can be stored as **database-generated columns**:

```sql
-- PostgreSQL example
ALTER TABLE "tbl_xxx" ADD COLUMN "fld_formula" NUMERIC
GENERATED ALWAYS AS (COALESCE("fld_price", 0) * COALESCE("fld_quantity", 0)) STORED;
```

**Supported for generated columns:**
- Arithmetic operations (+, -, *, /)
- String functions (CONCAT, UPPER, LOWER, etc.)
- Date extraction (YEAR, MONTH, DAY)
- Numeric functions (ROUND, ABS, etc.)

**NOT supported (requires application-level calculation):**
- NOW(), TODAY() (mutable functions)
- Link/Lookup field references
- Complex conditional logic
- Rollup aggregations

### 2.4 JSONB for Complex Types

Complex field types use JSONB storage:

```typescript
// Attachment field storage
[
  {
    "id": "att_xxx",
    "name": "document.pdf",
    "size": 102400,
    "mimetype": "application/pdf",
    "token": "abc123",
    "path": "attachments/2024/document.pdf"
  }
]

// Link field storage
[
  { "id": "rec_xxx", "title": "Linked Record 1" },
  { "id": "rec_yyy", "title": "Linked Record 2" }
]

// Multi-select storage
["Option A", "Option B", "Option C"]
```

---

## 3. Query Engine

### 3.1 Architecture Overview

```
RecordQueryBuilderService
    │
    ├── createQueryBuilder()     → Base Knex query
    ├── buildSelect()            → SELECT clause via FieldSelectVisitor
    ├── buildFilter()            → WHERE clause via FilterQuery
    └── buildSort()              → ORDER BY via SortQuery
```

### 3.2 Database Provider Interface

Located at: `apps/nestjs-backend/src/db-provider/db.provider.interface.ts`

```typescript
interface IDbProvider {
  driver: DriverClient;

  // Query builders
  filterQuery(qb, fields, filter, context): IFilterQueryInterface;
  sortQuery(qb, fields, sortObjs, context): ISortQueryInterface;
  aggregationQuery(qb, fields, aggregations, context): IAggregationQueryInterface;
  groupQuery(qb, fields, groupBy, context): IGroupQueryInterface;
  searchQuery(qb, fields, search): Knex.QueryBuilder;

  // Schema operations
  createSchema(schemaName): string[];
  dropTable(tableName): string;
  renameColumn(table, oldName, newName): string[];
  createColumnSchema(table, column, type, options): string[];
}
```

### 3.3 Filter Query System

**Filter Structure:**
```typescript
interface IFilter {
  filterSet: IFilterItem[];
  conjunction: 'and' | 'or';
}

interface IFilterItem {
  fieldId: string;
  operator: FilterOperator;
  value: any;
}
```

**Filter Operators by Field Type:**

| Field Type | Operators |
|------------|-----------|
| Text | is, isNot, contains, doesNotContain, isEmpty, isNotEmpty, startsWith, endsWith |
| Number | is, isNot, isGreater, isLess, isGreaterEqual, isLessEqual, isEmpty, isNotEmpty |
| Date | is, isNot, isBefore, isAfter, isOnOrBefore, isOnOrAfter, isWithin, isEmpty |
| Select | is, isNot, isAnyOf, isNoneOf, isEmpty, isNotEmpty |
| Checkbox | is |
| Link | contains, doesNotContain, isExactly, isEmpty, isNotEmpty |

**Filter Adapter Pattern:**

```typescript
// Each field type has a specific adapter
StringCellValueFilterAdapter   // Text fields
NumberCellValueFilterAdapter   // Numeric fields
DateTimeCellValueFilterAdapter // Date fields
BooleanCellValueFilterAdapter  // Checkbox
JsonCellValueFilterAdapter     // Multi-value fields
```

**Database-Specific SQL:**

```typescript
// PostgreSQL: Case-insensitive search
builderClient.whereRaw(`${column} iLIKE ?`, [`%${value}%`]);

// SQLite: Uses LOWER() function
builderClient.whereRaw(`LOWER(${column}) LIKE LOWER(?)`, [`%${value}%`]);
```

### 3.4 Sort Query System

```typescript
interface ISortItem {
  fieldId: string;
  order: 'asc' | 'desc';
}

// NULL handling
qb.orderByRaw(`${column} ASC NULLS FIRST`);   // ASC
qb.orderByRaw(`${column} DESC NULLS LAST`);   // DESC
```

### 3.5 Aggregation Operations

**Supported Functions:**
- `count`, `countEmpty`, `countFilled`, `countUnique`
- `sum`, `average`, `min`, `max`
- `percentEmpty`, `percentFilled`, `percentUnique`
- `dateRangeOfDays`, `dateRangeOfMonths`

```typescript
// Example: Sum aggregation
const sumQuery = knex.raw(`COALESCE(SUM(${column}), 0)`).toQuery();
```

---

## 4. Formula Engine

### 4.1 Architecture

```
packages/core/src/formula/
├── parser/                    # ANTLR4 grammar & generated code
│   ├── Formula.g4             # Parser grammar
│   └── FormulaLexer.g4        # Lexer grammar
├── functions/                 # 48+ built-in functions
├── visitor.ts                 # EvalVisitor (evaluator)
├── typed-value.ts             # Type system
└── field-reference.visitor.ts # Dependency extraction
```

### 4.2 Parser (ANTLR4)

**Grammar Highlights:**
```antlr
// Expressions
expr: literal
    | field_reference
    | function_call
    | expr BINARY_OP expr
    | UNARY_OP expr
    | '(' expr ')'
    ;

// Field references
field_reference: '{' FIELD_ID '}' ;

// Function calls
function_call: FUNC_NAME '(' expr (',' expr)* ')' ;
```

**Parsing Flow:**
```
Formula String → Lexer → Token Stream → Parser → Parse Tree (AST)
```

### 4.3 Function Library (48+ Functions)

**Numeric Functions (18):**
```
SUM, AVERAGE, MAX, MIN, ROUND, ROUNDUP, ROUNDDOWN, CEILING, FLOOR,
EVEN, ODD, INT, ABS, SQRT, POWER, EXP, LOG, MOD, VALUE
```

**Text Functions (15):**
```
CONCATENATE, FIND, SEARCH, MID, LEFT, RIGHT, REPLACE, REGEXP_REPLACE,
SUBSTITUTE, LOWER, UPPER, REPT, TRIM, LEN, T, ENCODE_URL_COMPONENT
```

**Logical Functions (9):**
```
IF, SWITCH, AND, OR, XOR, NOT, BLANK, ERROR, IS_ERROR
```

**Date/Time Functions (22):**
```
TODAY, NOW, YEAR, MONTH, WEEKNUM, WEEKDAY, DAY, HOUR, MINUTE, SECOND,
FROMNOW, TONOW, DATETIME_DIFF, WORKDAY, WORKDAY_DIFF, IS_SAME, IS_AFTER,
IS_BEFORE, DATE_ADD, DATESTR, TIMESTR, DATETIME_FORMAT, DATETIME_PARSE,
CREATED_TIME, LAST_MODIFIED_TIME
```

**Array Functions (7):**
```
COUNTALL, COUNTA, COUNT, ARRAY_JOIN, ARRAY_UNIQUE, ARRAY_FLATTEN,
ARRAY_COMPACT
```

**System Functions (3):**
```
TEXT_ALL, RECORD_ID, AUTO_NUMBER
```

### 4.4 Evaluation Strategy

**Two Modes:**

1. **On-Demand Evaluation** (default):
   - Formula evaluated in application code
   - Supports all functions including mutable ones (NOW, TODAY)
   - Used for complex formulas with link references

2. **Database-Generated Column** (optimized):
   - Formula converted to native SQL
   - Stored as `GENERATED ALWAYS AS` column
   - Automatically recalculated by database
   - Indexable for faster queries

**Generated Column Validator:**
```typescript
class FormulaSupportGeneratedColumnValidator {
  validateFormula(expression: string): boolean {
    // Check for unsupported functions
    if (containsMutableFunctions(expression)) return false;

    // Check for link field references
    if (hasLinkFieldReferences(expression)) return false;

    // Check all functions are SQL-convertible
    return allFunctionsSupported(expression);
  }
}
```

### 4.5 Dependency Tracking

**Circular Reference Detection:**
```typescript
class CircularReferenceError extends Error {
  fieldId: string;
  expansionStack: string[];

  getCircularChain(): string[] {
    // Returns: ["Field A", "Field B", "Field C", "Field A"]
  }
}
```

**Dependency Collector:**
```typescript
// Collects all field IDs referenced in a formula
FieldReferenceVisitor.getReferenceFieldIds(expression);
// Returns: ["fld_abc", "fld_xyz"]
```

### 4.6 Calculation Orchestration

Located at: `apps/nestjs-backend/src/features/record/computed/services/`

```typescript
// When a field value changes
async computeCellChangesForRecords(tableId, cellContexts, update) {
  // 1. Collect affected computed fields (dependency closure)
  const impact = await this.collector.collect(tableId, cellContexts);

  // 2. Execute the update transaction
  await update();

  // 3. Evaluate impacted records
  const publishedOps = await this.evaluator.evaluate(impact);

  return { publishedOps, impact };
}
```

---

## 5. Performance Optimization

### 5.1 Indexing Strategy

**Automatic Indexes:**
- Primary key (`__id`)
- Foreign keys for link fields
- `dbTableName` on TableMeta (lookup optimization)

**Recommended User Indexes:**
- Fields used in filters
- Fields used in sorts
- Unique constraints

### 5.2 Query Optimization

**Pagination via CTE:**
```sql
WITH base_query AS (
  SELECT * FROM "tbl_xxx"
  WHERE /* filters */
  ORDER BY /* sorts */
  LIMIT /* offset + limit */
)
SELECT * FROM base_query;
```

**Projection Support:**
- Only select requested fields
- Reduces data transfer
- Improves query performance

### 5.3 Caching Layers

1. **Query Result Cache** (Redis):
   - Key: `record:{tableId}:{version}:{queryHash}`
   - TTL: Configurable

2. **Computed Field Cache**:
   - Cached at application level
   - Invalidated on dependency change

3. **Connection Pooling**:
   - Knex connection pool
   - Configured via environment variables

### 5.4 Large Dataset Handling

**Batch Processing:**
```typescript
// Records processed in configurable chunks
const chunkSize = calcChunkSize(recordCount);
for (let i = 0; i < records.length; i += chunkSize) {
  await processChunk(records.slice(i, i + chunkSize));
}
```

**Streaming Exports:**
- CSV/Excel exports use streaming
- 1000 records per query batch
- Memory-efficient for large datasets

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| Prisma Schema | `packages/db-main-prisma/prisma/template.prisma` |
| Database Providers | `apps/nestjs-backend/src/db-provider/` |
| Filter Adapters | `apps/nestjs-backend/src/db-provider/filter-query/` |
| Sort Adapters | `apps/nestjs-backend/src/db-provider/sort-query/` |
| Formula Parser | `packages/core/src/formula/parser/` |
| Formula Functions | `packages/core/src/formula/functions/` |
| Formula Evaluator | `packages/core/src/formula/visitor.ts` |
| Query Builder | `apps/nestjs-backend/src/features/record/query-builder/` |
| Computed Fields | `apps/nestjs-backend/src/features/record/computed/` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Real-time Architecture](./ARCHITECTURE-REALTIME.md)*
