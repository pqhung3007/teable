# Teable Testing & Observability Architecture

> **Detailed documentation of testing strategy, logging, tracing, and monitoring.**

## Table of Contents

1. [Testing Strategy](#1-testing-strategy)
2. [Unit Testing](#2-unit-testing)
3. [Integration Testing](#3-integration-testing)
4. [E2E Testing](#4-e2e-testing)
5. [Logging](#5-logging)
6. [Distributed Tracing](#6-distributed-tracing)
7. [Error Monitoring](#7-error-monitoring)
8. [Health Checks](#8-health-checks)

---

## 1. Testing Strategy

### 1.1 Test Framework

Teable uses **Vitest** as the primary test runner across the monorepo:

| Framework | Version | Purpose |
|-----------|---------|---------|
| Vitest | 2.1.5 | Unit & integration tests |
| Playwright | 1.42.1 | E2E browser tests |
| Testing Library | 14.2.2 | React component tests |
| vitest-mock-extended | 2.0.2 | Deep mocking |

### 1.2 Test Organization

```
apps/nestjs-backend/
├── src/**/*.spec.ts           # Unit tests (colocated)
├── test/**/*.e2e-spec.ts      # Integration tests
└── test/utils/                # Test utilities

apps/nextjs-app/
├── src/**/__tests__/*.test.tsx  # Unit tests
├── e2e/**/*.spec.ts             # Playwright E2E tests
└── config/tests/                # Test configuration

packages/core/
└── src/**/*.spec.ts           # Package unit tests
```

### 1.3 Test Commands

```bash
# Root commands (all packages)
pnpm g:test-unit              # Run all unit tests
pnpm g:test-unit-cover        # With coverage
pnpm g:test-e2e               # Run all E2E tests

# Backend specific
cd apps/nestjs-backend
pnpm test-unit                # Unit tests
pnpm test-e2e                 # Integration tests (requires setup)
pnpm merge-cover              # Merge coverage reports

# Frontend specific
cd apps/nextjs-app
pnpm test                     # Unit tests
pnpm e2e                      # Playwright tests
```

---

## 2. Unit Testing

### 2.1 Configuration

**Backend** (`apps/nestjs-backend/vitest.config.ts`):
```typescript
export default defineConfig({
  test: {
    environment: 'node',
    include: ['**/src/**/*.{test,spec}.{js,ts}'],
    exclude: ['**/*.controller.spec.ts'],  // Integration tests
    cache: { dir: '.cache/vitest/unit' },
    pool: 'forks',
    poolOptions: { forks: { singleFork: true } },
    coverage: {
      provider: 'v8',
      reportsDirectory: './coverage/unit'
    }
  }
});
```

**Frontend** (`apps/nextjs-app/vitest.config.ts`):
```typescript
export default defineConfig({
  test: {
    environment: 'happy-dom',
    include: ['./src/**/*.{test,spec}.{js,jsx,ts,tsx}'],
    setupFiles: ['./config/tests/setupVitest.ts'],
    coverage: {
      provider: 'v8'
    }
  }
});
```

### 2.2 Service Testing Pattern

```typescript
// apps/nestjs-backend/src/features/base/base.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { mockDeep } from 'vitest-mock-extended';

describe('BaseService', () => {
  let service: BaseService;
  let prisma: DeepMockProxy<PrismaService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BaseService,
        {
          provide: PrismaService,
          useValue: mockDeep<PrismaService>()
        }
      ]
    }).compile();

    service = module.get(BaseService);
    prisma = module.get(PrismaService);
  });

  it('should create a base', async () => {
    // Arrange
    const createBaseDto = { name: 'Test Base', spaceId: 'sp_123' };
    prisma.base.create.mockResolvedValue({ id: 'bas_123', ...createBaseDto });

    // Act
    const result = await service.createBase(createBaseDto);

    // Assert
    expect(result.id).toBe('bas_123');
    expect(prisma.base.create).toHaveBeenCalledWith({
      data: expect.objectContaining({ name: 'Test Base' })
    });
  });
});
```

### 2.3 Utility Testing Pattern

```typescript
// packages/core/src/formula/visitor.spec.ts
import { describe, it, expect } from 'vitest';
import { evaluate } from './visitor';

describe('Formula Evaluator', () => {
  it('should evaluate SUM function', () => {
    const result = evaluate('SUM({field1}, {field2})', {
      field1: createNumberField(10),
      field2: createNumberField(20)
    }, mockRecord);

    expect(result.value).toBe(30);
    expect(result.type).toBe(CellValueType.Number);
  });

  it('should handle circular references', () => {
    expect(() => evaluate('{self}', { self: selfReferencingField }))
      .toThrow(CircularReferenceError);
  });
});
```

### 2.4 React Component Testing

```typescript
// apps/nextjs-app/src/components/__tests__/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from '../Button';

describe('Button', () => {
  it('should render with label', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button')).toHaveTextContent('Click me');
  });

  it('should call onClick handler', () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click</Button>);

    fireEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledOnce();
  });
});
```

---

## 3. Integration Testing

### 3.1 Configuration

**E2E Config** (`apps/nestjs-backend/vitest-e2e.config.ts`):
```typescript
export default defineConfig({
  test: {
    include: ['**/test/**/*.{e2e-test,e2e-spec}.{js,ts}'],
    setupFiles: ['./vitest-e2e.setup.ts'],
    testTimeout: process.env.CI ? 30000 : 10000,
    hookTimeout: 30000,
    pool: 'forks',
    poolOptions: { forks: { singleFork: true } }
  }
});
```

### 3.2 Test Setup

**Setup File** (`vitest-e2e.setup.ts`):
```typescript
import { beforeAll, afterAll } from 'vitest';
import esbuild from 'esbuild';

beforeAll(async () => {
  // Set test environment
  process.env.NODE_ENV = 'test';
  process.env.LOG_LEVEL = 'error';

  // Configure database
  if (process.env.TEST_DRIVER === 'sqlite') {
    process.env.PRISMA_DATABASE_URL = 'file:.assets/test.db';
  }

  // Build worker files
  await esbuild.build({
    entryPoints: ['src/workers/*.ts'],
    outdir: 'dist/workers',
    bundle: true
  });
});
```

### 3.3 App Initialization

**Location**: `apps/nestjs-backend/test/utils/init-app.ts`

```typescript
export async function initApp() {
  // Create test module
  const moduleRef = await Test.createTestingModule({
    imports: [AppModule]
  })
    .overrideProvider(NextService).useValue({})
    .overrideProvider(DevWsGateway).useValue({})
    .compile();

  // Create application
  const app = moduleRef.createNestApplication();
  await app.init();
  await app.listen(0);  // Random port

  // Get auth cookie
  const cookie = await login(app);

  return {
    app,
    url: await app.getUrl(),
    cookie,
    // Helper methods
    createSpace: () => createSpace(app, cookie),
    createBase: (spaceId) => createBase(app, cookie, spaceId),
    createTable: (baseId) => createTable(app, cookie, baseId),
    createField: (tableId, field) => createField(app, cookie, tableId, field),
    createRecords: (tableId, records) => createRecords(app, cookie, tableId, records)
  };
}
```

### 3.4 Integration Test Pattern

```typescript
// apps/nestjs-backend/test/record.e2e-spec.ts
import { describe, beforeAll, afterAll, it, expect } from 'vitest';
import { initApp } from './utils/init-app';

describe('Record API (e2e)', () => {
  let app: INestApplication;
  let cookie: string;
  let tableId: string;

  beforeAll(async () => {
    const testApp = await initApp();
    app = testApp.app;
    cookie = testApp.cookie;

    // Setup test data
    const space = await testApp.createSpace();
    const base = await testApp.createBase(space.id);
    const table = await testApp.createTable(base.id);
    tableId = table.id;
  });

  afterAll(async () => {
    await app.close();
  });

  it('should create a record', async () => {
    const response = await request(app.getHttpServer())
      .post(`/api/tables/${tableId}/records`)
      .set('Cookie', cookie)
      .send({ records: [{ fields: { Name: 'Test' } }] });

    expect(response.status).toBe(201);
    expect(response.body.records[0].fields.Name).toBe('Test');
  });

  it('should filter records', async () => {
    const response = await request(app.getHttpServer())
      .get(`/api/tables/${tableId}/records`)
      .set('Cookie', cookie)
      .query({
        filter: JSON.stringify({
          conjunction: 'and',
          filterSet: [{ fieldId: 'fld_name', operator: 'contains', value: 'Test' }]
        })
      });

    expect(response.status).toBe(200);
    expect(response.body.records.length).toBeGreaterThan(0);
  });
});
```

### 3.5 Database Matrix Testing

```yaml
# .github/workflows/integration-tests.yml
jobs:
  test:
    strategy:
      matrix:
        database: [sqlite, postgres]
    steps:
      - name: Run tests
        env:
          TEST_DRIVER: ${{ matrix.database }}
        run: pnpm test-e2e
```

---

## 4. E2E Testing

### 4.1 Playwright Configuration

**Location**: `apps/nextjs-app/playwright.config.ts`

```typescript
export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: process.env.CI ? 3 : 1,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['json', { outputFile: 'playwright-report.json' }],
    ['html', { open: 'never' }]
  ],
  use: {
    trace: 'retry-with-trace',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['Pixel 5'] } }
  ],
  webServer: {
    command: 'pnpm start',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
});
```

### 4.2 E2E Test Pattern

```typescript
// apps/nextjs-app/e2e/table/create-table.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Table Creation', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/auth/login');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');
    await page.waitForURL('/space/*');
  });

  test('should create a new table', async ({ page }) => {
    // Navigate to base
    await page.click('[data-testid="base-link"]');
    await page.waitForURL('/base/*');

    // Create table
    await page.click('[data-testid="create-table-button"]');
    await page.fill('[data-testid="table-name-input"]', 'My Table');
    await page.click('[data-testid="confirm-create"]');

    // Verify
    await expect(page.locator('[data-testid="table-header"]'))
      .toHaveText('My Table');
  });
});
```

---

## 5. Logging

### 5.1 Configuration

**Location**: `apps/nestjs-backend/src/configs/logger.config.ts`

```typescript
export const loggerConfig = registerAs('logger', () => ({
  level: process.env.LOG_LEVEL ?? 'info',
  enableGlobalErrorLogging: process.env.ENABLE_GLOBAL_ERROR_LOGGING === 'true'
}));

// Supported levels: fatal, error, warn, info, debug, trace
```

### 5.2 Pino Setup

**Location**: `apps/nestjs-backend/src/logger/logger.module.ts`

```typescript
@Module({})
export class LoggerModule {
  static register() {
    return LoggerModule.forRoot({
      pinoHttp: {
        level: config.level,
        transport: isDevelopment ? {
          target: 'pino-pretty',
          options: { colorize: true }
        } : undefined,

        // Exclude noisy routes
        autoLogging: {
          ignore: (req) =>
            req.url.includes('/health') ||
            req.url.includes('/_next/') ||
            req.url.includes('/favicon.ico')
        },

        // Request ID from OpenTelemetry
        genReqId: (req, res) => {
          const span = trace.getActiveSpan();
          return span?.spanContext().traceId ?? nanoid();
        },

        // Custom log formatters
        customProps: (req) => ({
          traceId: trace.getActiveSpan()?.spanContext().traceId,
          spanId: trace.getActiveSpan()?.spanContext().spanId
        })
      }
    });
  }
}
```

### 5.3 Log Output Example

```json
{
  "level": 30,
  "time": 1699999999999,
  "pid": 12345,
  "hostname": "teable-1",
  "traceId": "abc123def456",
  "spanId": "xyz789",
  "req": {
    "id": "abc123def456",
    "method": "GET",
    "url": "/api/tables/tbl_123/records",
    "headers": { "user-agent": "..." }
  },
  "res": {
    "statusCode": 200
  },
  "responseTime": 45,
  "msg": "request completed"
}
```

---

## 6. Distributed Tracing

### 6.1 OpenTelemetry Setup

**Location**: `apps/nestjs-backend/src/tracing.ts`

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME ?? 'teable',
  serviceVersion: process.env.BUILD_VERSION,
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
    new NestInstrumentation(),
    new PrismaInstrumentation(),
    new PinoInstrumentation()
  ],
  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBasedSampler(
      parseFloat(process.env.OTEL_SAMPLER_RATIO ?? '0.1')
    )
  })
});

sdk.start();
```

### 6.2 Route Tracing Interceptor

**Location**: `apps/nestjs-backend/src/tracing/route-tracing.interceptor.ts`

```typescript
@Injectable()
export class RouteTracingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const span = trace.getActiveSpan();

    if (span) {
      const request = context.switchToHttp().getRequest();
      const route = this.getRoute(request);

      span.setAttributes({
        'http.method': request.method,
        'http.route': route,
        'nest.controller': context.getClass().name,
        'nest.handler': context.getHandler().name
      });

      span.updateName(`${request.method} ${route}`);
    }

    return next.handle().pipe(
      tap(() => {
        span?.setAttributes({
          'http.status_code': response.statusCode
        });
      })
    );
  }
}
```

### 6.3 Custom Span Decorator

```typescript
// Usage
@Span('processRecords')
async processRecords(records: IRecord[]) {
  // Method is automatically traced
}

// Implementation
function Span(name?: string) {
  return (target, propertyKey, descriptor) => {
    const original = descriptor.value;

    descriptor.value = async function(...args) {
      const tracer = trace.getTracer('teable');
      const spanName = name || `${target.constructor.name}.${propertyKey}`;

      return tracer.startActiveSpan(spanName, async (span) => {
        try {
          const result = await original.apply(this, args);
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
        } catch (error) {
          span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
          span.recordException(error);
          throw error;
        } finally {
          span.end();
        }
      });
    };
  };
}
```

### 6.4 Configuration

```bash
# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer token
OTEL_SERVICE_NAME=teable
OTEL_SAMPLER_RATIO=0.1  # Sample 10% of requests
BUILD_VERSION=1.0.0
```

---

## 7. Error Monitoring

### 7.1 Sentry Integration

**Backend** (`apps/nestjs-backend/src/instrument.ts`):

```typescript
import * as Sentry from '@sentry/nestjs';

Sentry.init({
  dsn: process.env.BACKEND_SENTRY_DSN,
  tracesSampleRate: parseFloat(process.env.BACKEND_SENTRY_TRACE_SAMPLING_RATE ?? '0.1'),
  integrations: [
    consoleIntegration({ levels: ['warn', 'error'] }),
    httpIntegration(),
    nestIntegration(),
    prismaIntegration()
  ]
});
```

**Frontend** (`apps/nextjs-app/sentry.client.config.ts`):

```typescript
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [new Sentry.Replay()]
});
```

### 7.2 Global Exception Filter

**Location**: `apps/nestjs-backend/src/filter/global-exception.filter.ts`

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  @SentryExceptionCaptured()
  catch(exception: Error, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    const request = host.switchToHttp().getRequest();

    // Log error (unless it's a known client error)
    if (!(exception instanceof BadRequestException ||
          exception instanceof UnauthorizedException)) {
      this.logger.error({
        message: exception.message,
        stack: exception.stack,
        path: request.url,
        method: request.method
      });
    }

    // Parse and normalize exception
    const httpException = this.parseException(exception);

    return response.status(httpException.status).json({
      message: httpException.message,
      status: httpException.status,
      code: httpException.code,
      data: httpException.data
    });
  }
}
```

---

## 8. Health Checks

### 8.1 Backend Health Endpoints

**Location**: `apps/nestjs-backend/src/features/health/health.controller.ts`

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prisma: PrismaHealthIndicator
  ) {}

  @Get()
  @Public()
  async check() {
    return this.health.check([
      () => this.prisma.pingCheck('database')
    ]);
  }

  @Get('memory')
  @Public()
  getMemory() {
    const memUsage = process.memoryUsage();
    return {
      hostname: os.hostname(),
      heapUsed: memUsage.heapUsed,
      heapTotal: memUsage.heapTotal,
      external: memUsage.external,
      rss: memUsage.rss
    };
  }
}
```

### 8.2 Health Check Response

```json
// GET /health
{
  "status": "ok",
  "info": {
    "database": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" }
  }
}

// GET /health/memory
{
  "hostname": "teable-pod-abc123",
  "heapUsed": 125829120,
  "heapTotal": 167772160,
  "external": 10485760,
  "rss": 209715200
}
```

### 8.3 Frontend Health Check

**Location**: `apps/nextjs-app/src/pages/api/_monitor/healthcheck.ts`

```typescript
export default function handler(req, res) {
  res.status(200).json({
    status: 'ok',
    message: 'Healthy',
    appName: 'teable-web',
    appVersion: process.env.BUILD_VERSION,
    timestamp: new Date().toISOString()
  });
}
```

---

## CI/CD Integration

### GitHub Actions Workflows

**Unit Tests** (`.github/workflows/unit-tests.yml`):
```yaml
name: Unit Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: pnpm install
      - run: pnpm g:build
      - run: pnpm g:test-unit
```

**Integration Tests** (`.github/workflows/integration-tests.yml`):
```yaml
name: Integration Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install
      - run: pnpm test-e2e
        env:
          TEST_DRIVER: postgres
          DATABASE_URL: postgres://postgres:postgres@localhost/test
```

---

## Key Configuration Summary

| Feature | Environment Variable | Default |
|---------|---------------------|---------|
| Log Level | `LOG_LEVEL` | `info` |
| Error Logging | `ENABLE_GLOBAL_ERROR_LOGGING` | `false` |
| Sentry DSN | `BACKEND_SENTRY_DSN` | - |
| Sentry Sample Rate | `BACKEND_SENTRY_TRACE_SAMPLING_RATE` | `0.1` |
| OTEL Endpoint | `OTEL_EXPORTER_OTLP_ENDPOINT` | - |
| OTEL Sample Rate | `OTEL_SAMPLER_RATIO` | `0.1` |
| Service Name | `OTEL_SERVICE_NAME` | `teable` |

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| Vitest Config (Backend) | `apps/nestjs-backend/vitest.config.ts` |
| Vitest E2E Config | `apps/nestjs-backend/vitest-e2e.config.ts` |
| Playwright Config | `apps/nextjs-app/playwright.config.ts` |
| Test Utils | `apps/nestjs-backend/test/utils/` |
| Logger Module | `apps/nestjs-backend/src/logger/logger.module.ts` |
| Tracing Setup | `apps/nestjs-backend/src/tracing.ts` |
| Sentry Setup | `apps/nestjs-backend/src/instrument.ts` |
| Health Controller | `apps/nestjs-backend/src/features/health/` |
| Exception Filter | `apps/nestjs-backend/src/filter/global-exception.filter.ts` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Operations Architecture](./ARCHITECTURE-OPERATIONS.md)*
