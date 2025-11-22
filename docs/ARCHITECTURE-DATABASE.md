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

### Overview

The database schema is the foundation of Teable's architecture. Unlike traditional applications where the schema is fixed at development time, Teable must support user-defined tables and fields created at runtime. This presents unique challenges:

- **Dynamic structure**: Users create tables and fields without writing SQL
- **Type safety**: Each field type needs appropriate database representation
- **Performance**: Dynamically created tables must still support fast queries
- **Multi-tenancy**: Multiple workspaces share the same database safely

The solution is a **metadata-driven approach** where fixed "metadata tables" (managed by Prisma) describe dynamic "data tables" (created via raw SQL at runtime).

### 1.1 Core Model Hierarchy

Understanding the containment hierarchy is essential for navigating the codebase. Each level owns the resources below it:

```
Space (workspace)                   # Top-level container, billing unit
└── Base (database container)       # Logical grouping of related tables
    └── TableMeta (table definition)# Metadata about a user table
        ├── Field (column definitions)  # What columns the table has
        ├── View (view configurations)  # Saved filter/sort/grouping
        └── Records (via physical table)# Actual data rows
```

**Why this hierarchy?**
- **Space** provides workspace-level isolation and is the unit for access control
- **Base** groups related tables (like a project or department)
- **TableMeta** bridges between the metadata world and physical data tables
- **Field/View** are configuration stored in metadata, not in user data

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

Multi-tenancy—serving multiple customers from a single deployment—is implemented through **workspace-based isolation** rather than database-per-tenant. This approach:

- **Simplifies operations**: One database to backup, monitor, and scale
- **Enables resource sharing**: Efficient for small workspaces that don't need dedicated resources
- **Supports collaboration**: Users can be members of multiple workspaces

The access path for any request:
```
User → AccessToken/Session → Collaborator → Space/Base
```

**Permission enforcement** happens at the `Collaborator` level:
- A `Collaborator` record links a user to a Space or Base with a specific role
- Every API request checks collaborator permissions before accessing data
- **Access Tokens** can be scoped to specific spaces/bases for API integrations

**No explicit Organization model** - Teable keeps the model simple:
- **Space** is the top-level container (effectively an "organization")
- Multiple spaces can belong to the same user
- Billing and quotas are tracked at the Space level

---

## 2. Dynamic Field Storage

### Overview

Dynamic field storage is what enables Teable's no-code flexibility. When a user adds a "Number" column to their table, Teable must:
1. Update the metadata (Field table) to record the new column's configuration
2. Execute DDL (ALTER TABLE) to add the physical column to the data table
3. Handle the type conversion between UI values and database storage

This section explains how fields are physically represented in the database.

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

### Overview

The query engine translates high-level record queries into optimized SQL. This is challenging because:

- **Dynamic schemas**: The query engine doesn't know table structures at compile time
- **Multiple databases**: SQL must work for both PostgreSQL and SQLite
- **Complex filtering**: 40+ operators across different field types require type-specific SQL
- **Performance**: Queries on large tables must use indexes effectively

Teable uses **Knex.js** as a query builder, with a custom abstraction layer (`DbProvider`) that handles database-specific differences.

### 3.1 Architecture Overview

The query building process follows a pipeline pattern where each stage adds to the query:

```
RecordQueryBuilderService
    │
    ├── createQueryBuilder()     → Base Knex query with table reference
    ├── buildSelect()            → SELECT clause via FieldSelectVisitor
    │                              (handles computed fields, type conversions)
    ├── buildFilter()            → WHERE clause via FilterQuery
    │                              (type-specific operators, nested conditions)
    └── buildSort()              → ORDER BY via SortQuery
                                   (NULL handling, multi-column sort)
```

**Why Knex instead of Prisma for queries?** Prisma is great for typed CRUD on known schemas, but Teable's dynamic tables require raw SQL flexibility. Knex provides a nice middle ground—programmatic query building with database abstraction.

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

The filter system is where database abstraction becomes most complex. Each combination of field type and operator requires specific SQL generation, and that SQL differs between PostgreSQL and SQLite.

**Filter Structure:**
```typescript
interface IFilter {
  filterSet: IFilterItem[];    // Array of conditions
  conjunction: 'and' | 'or';   // How to combine them
}

interface IFilterItem {
  fieldId: string;             // Which field to filter
  operator: FilterOperator;    // Which comparison to use
  value: any;                  // Value to compare against
}
```

**Filter Operators by Field Type:**

Different field types support different operators. For example, "contains" makes sense for text but not for numbers.

| Field Type | Operators | Notes |
|------------|-----------|-------|
| Text | is, isNot, contains, doesNotContain, isEmpty, isNotEmpty, startsWith, endsWith | Case-insensitive by default |
| Number | is, isNot, isGreater, isLess, isGreaterEqual, isLessEqual, isEmpty, isNotEmpty | NULL handling is critical |
| Date | is, isNot, isBefore, isAfter, isOnOrBefore, isOnOrAfter, isWithin, isEmpty | Timezone-aware comparisons |
| Select | is, isNot, isAnyOf, isNoneOf, isEmpty, isNotEmpty | Exact string matching |
| Checkbox | is | True/false only |
| Link | contains, doesNotContain, isExactly, isEmpty, isNotEmpty | JSONB array operations |

**Filter Adapter Pattern:**

The Adapter pattern isolates database-specific logic. Each adapter knows how to generate SQL for its field type:

```typescript
// Each field type has a specific adapter
StringCellValueFilterAdapter   // Text fields → VARCHAR operators
NumberCellValueFilterAdapter   // Numeric fields → arithmetic comparisons
DateTimeCellValueFilterAdapter // Date fields → temporal operators
BooleanCellValueFilterAdapter  // Checkbox → boolean logic
JsonCellValueFilterAdapter     // Multi-value fields → JSONB/array operators
```

**Database-Specific SQL:**

The same logical operation ("contains") produces different SQL:

```typescript
// PostgreSQL: Native case-insensitive ILIKE operator
builderClient.whereRaw(`${column} iLIKE ?`, [`%${value}%`]);

// SQLite: No ILIKE, must use LOWER() function
builderClient.whereRaw(`LOWER(${column}) LIKE LOWER(?)`, [`%${value}%`]);
```

This abstraction means filter logic is written once, but correct SQL is generated for each database.

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

### Overview

The formula engine enables Excel-like calculated fields in Teable. Users write expressions like `{Price} * {Quantity}` or `IF({Status} = "Complete", "✓", "")`, and the engine evaluates them automatically when referenced fields change.

Building a formula engine requires solving several problems:
- **Parsing**: Convert text expressions into executable form
- **Type system**: Handle type coercion between different field types
- **Evaluation**: Calculate results efficiently, either in the application or database
- **Dependencies**: Track which fields a formula depends on for recalculation
- **Circular detection**: Prevent formulas that reference themselves

Teable uses **ANTLR4** for parsing—the same tool used by SQL databases and programming languages—providing a robust foundation for expression handling.

### 4.1 Architecture

The formula system is organized as a pipeline from text to value:

```
packages/core/src/formula/
├── parser/                    # ANTLR4 grammar & generated code
│   ├── Formula.g4             # Parser grammar (expression syntax)
│   └── FormulaLexer.g4        # Lexer grammar (token definitions)
├── functions/                 # 48+ built-in functions (SUM, IF, TODAY, etc.)
├── visitor.ts                 # EvalVisitor (tree-walking evaluator)
├── typed-value.ts             # Type system (numbers, strings, dates, arrays)
└── field-reference.visitor.ts # Dependency extraction for recalculation
```

**Why ANTLR4?** ANTLR generates efficient parsers from grammar definitions. This ensures:
- Correct handling of operator precedence and associativity
- Clear error messages for syntax errors
- Maintainable grammar as features are added

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

One of the most important architectural decisions is **where** formulas are evaluated. Teable supports two modes, chosen automatically based on formula complexity:

**Two Modes:**

| Mode | How It Works | When Used | Trade-offs |
|------|--------------|-----------|------------|
| **On-Demand (Application)** | Formula evaluated in Node.js when records are fetched | Complex formulas, NOW/TODAY, link references | Flexible but slower for large datasets |
| **Database-Generated Column** | Formula converted to SQL GENERATED column | Simple arithmetic, string ops | Fast and indexable, but limited functions |

**1. On-Demand Evaluation** (default):
- Formula evaluated in application code using the EvalVisitor
- Supports **all functions** including mutable ones (NOW, TODAY change on each call)
- Required for formulas referencing **link fields** (needs JOIN logic)
- Values computed when records are queried, not stored

**2. Database-Generated Column** (optimized):
- Formula converted to native SQL and stored as `GENERATED ALWAYS AS` column
- The database automatically recalculates when source columns change
- Can be **indexed** for faster filtering/sorting on computed values
- Limited to functions that have SQL equivalents

**How Teable decides which mode to use:**

```typescript
class FormulaSupportGeneratedColumnValidator {
  validateFormula(expression: string): boolean {
    // Mutable functions (NOW, TODAY) can't be generated columns
    // because their values change even when data doesn't
    if (containsMutableFunctions(expression)) return false;

    // Link field references require JOINs that generated columns can't do
    if (hasLinkFieldReferences(expression)) return false;

    // All functions in the expression must have SQL equivalents
    return allFunctionsSupported(expression);
  }
}
```

**Example: Generated column vs on-demand**
- `{Price} * {Quantity}` → Generated column (simple arithmetic)
- `{Price} * {Quantity} * (1 - {Discount})` → Generated column
- `DAYS_BETWEEN(TODAY(), {DueDate})` → On-demand (TODAY is mutable)
- `{LinkedOrder}.{Total}` → On-demand (requires link resolution)

### 4.5 Dependency Tracking

When a field value changes, all formulas that reference it must be recalculated. Teable maintains a **dependency graph** to track these relationships efficiently.

**Why dependency tracking matters:**
- Changing a `Price` field should update `Total = Price * Quantity`
- But it should NOT recalculate unrelated formulas
- The graph enables efficient invalidation without full-table scans

**Circular Reference Detection:**

Formulas can create cycles: A references B, B references C, C references A. These are detected at field creation time:

```typescript
class CircularReferenceError extends Error {
  fieldId: string;           // The field that would create the cycle
  expansionStack: string[];  // The chain of dependencies

  getCircularChain(): string[] {
    // Returns: ["Field A", "Field B", "Field C", "Field A"]
    // Shows exactly where the cycle occurs
  }
}
```

**Dependency Collector:**

The `FieldReferenceVisitor` walks the formula AST to extract all field references:

```typescript
// Collects all field IDs referenced in a formula
const refs = FieldReferenceVisitor.getReferenceFieldIds(expression);
// For "{Price} * {Quantity}", returns: ["fld_price", "fld_quantity"]

// These are stored in the Reference table for efficient lookup
await prisma.reference.createMany({
  data: refs.map(fromFieldId => ({
    toFieldId: formulaFieldId,    // The formula field
    fromFieldId: fromFieldId,     // A field it depends on
  }))
});
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

### Overview

Performance is critical for a database platform. Users expect:
- **Fast page loads**: Record lists should appear instantly
- **Responsive filtering**: Filter changes should feel immediate
- **Scalable growth**: Performance shouldn't degrade as data grows

This section covers the strategies Teable uses to maintain performance across different scales.

### 5.1 Indexing Strategy

Indexes are the primary tool for query performance. Teable creates some indexes automatically and provides guidance for user-created indexes.

**Automatic Indexes:**
- **Primary key** (`__id`) - Every record lookup is fast
- **Foreign keys** for link fields - JOIN operations are optimized
- **`dbTableName`** on TableMeta - Table lookup by physical name is instant
- **Auto-number** (`__auto_number`) - Default sorting is efficient

**When to add custom indexes:**
- Fields frequently used in filters (WHERE clauses)
- Fields used for sorting (ORDER BY)
- Fields with unique constraints (also serves as index)
- Fields used in link field "lookup by" queries

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
