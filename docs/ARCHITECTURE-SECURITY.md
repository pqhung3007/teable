# Teable Security Architecture

> **Detailed documentation of authentication, authorization, and access control.**

## Table of Contents

1. [Authentication Overview](#1-authentication-overview)
2. [Session Management](#2-session-management)
3. [JWT & Access Tokens](#3-jwt--access-tokens)
4. [OAuth Integration](#4-oauth-integration)
5. [Authorization (RBAC)](#5-authorization-rbac)
6. [Permission Model](#6-permission-model)
7. [Guards & Decorators](#7-guards--decorators)
8. [Security Best Practices](#8-security-best-practices)

---

## 1. Authentication Overview

### 1.1 Multi-Strategy Authentication

Teable supports multiple authentication strategies:

| Strategy | Use Case | Header/Cookie |
|----------|----------|---------------|
| **Session** | Web UI | `TEABLE_SID` cookie |
| **Access Token** | API calls | `Authorization: Bearer teable_...` |
| **JWT** | Internal services | `Authorization: Bearer <jwt>` |

### 1.2 Authentication Flow

```
Request
    │
    ▼
┌──────────────────────────────────────┐
│            AuthGuard                  │
│  ┌────────────────────────────────┐  │
│  │ Try Strategy 1: Session        │  │
│  │ Cookie → SessionStore → User   │  │
│  └────────────┬───────────────────┘  │
│               │ (if fails)           │
│  ┌────────────▼───────────────────┐  │
│  │ Try Strategy 2: Access Token   │  │
│  │ Bearer → Decrypt → Validate    │  │
│  └────────────┬───────────────────┘  │
│               │ (if fails)           │
│  ┌────────────▼───────────────────┐  │
│  │ Try Strategy 3: JWT            │  │
│  │ Bearer → Verify → Extract      │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
    │
    ▼
User Authenticated (or 401)
```

### 1.3 Auth Module Structure

```
apps/nestjs-backend/src/features/auth/
├── auth.module.ts          # Main module
├── auth.service.ts         # Token generation
├── auth.controller.ts      # Auth endpoints
├── permission.module.ts    # Permission checking
├── permission.service.ts   # RBAC logic
├── guard/
│   ├── auth.guard.ts       # Multi-strategy guard
│   └── permission.guard.ts # Authorization guard
├── strategies/
│   ├── session.strategy.ts
│   ├── access-token.strategy.ts
│   └── jwt.strategy.ts
├── session/
│   ├── session-handle.service.ts
│   └── session-store.service.ts
├── local-auth/             # Password auth
└── social/                 # OAuth providers
```

---

## 2. Session Management

### 2.1 Session Configuration

```typescript
// Environment variables
BACKEND_SESSION_SECRET=<your-secret>
BACKEND_SESSION_EXPIRES_IN=7d

// Session cookie
Cookie name: TEABLE_SID
Secure: auto|true|false (configurable)
HttpOnly: true
SameSite: lax
```

### 2.2 Session Store

**Location**: `apps/nestjs-backend/src/features/auth/session/session-store.service.ts`

```typescript
export class SessionStoreService extends Store {
  // Store session data
  async set(sid: string, session: ISessionData, callback) {
    await this.cacheService.set(`auth:session-store:${sid}`, session, this.ttl);

    // Track user's sessions for bulk logout
    const userSessions = await this.cacheService.get(`auth:session-user:${userId}`);
    userSessions[sid] = expirationTime;
    await this.cacheService.set(`auth:session-user:${userId}`, userSessions);
  }

  // Clear all user sessions (logout everywhere)
  async clearByUserId(userId: string) {
    const userSessions = await this.cacheService.get(`auth:session-user:${userId}`);
    for (const sid of Object.keys(userSessions)) {
      await this.cacheService.del(`auth:session-store:${sid}`);
    }
    await this.cacheService.del(`auth:session-user:${userId}`);
  }
}
```

### 2.3 Session Serialization

```typescript
// Serialize user for session storage
serializeUser(user, done) {
  done(null, { id: user.id });
}

// Deserialize user from session
deserializeUser(payload, done) {
  const user = await this.userService.getUserById(payload.id);
  done(null, user);
}
```

---

## 3. JWT & Access Tokens

### 3.1 JWT Configuration

```typescript
// Environment variables
BACKEND_JWT_SECRET=<your-secret>
BACKEND_JWT_EXPIRES_IN=20d
```

### 3.2 JWT Types

**User JWT:**
```typescript
{
  userId: string;
  exp: number;
  iat: number;
}
```

**Internal JWT (Automation/Apps):**
```typescript
{
  type: 'automation' | 'app';
  baseId: string;
  exp: number;
  iat: number;
}
```

### 3.3 Access Token System

**Database Model:**
```prisma
model AccessToken {
  id            String    @id
  name          String
  description   String?
  userId        String
  scopes        String    // JSON array of Action[]
  spaceIds      String?   // JSON array - restrict to spaces
  baseIds       String?   // JSON array - restrict to bases
  sign          String    // 16-char random signature
  hasFullAccess Boolean?
  expiredTime   DateTime
  lastUsedTime  DateTime?
  createdTime   DateTime
}
```

**Token Format:**
```
teable_<tokenId>_<encryptedSign>

Example: teable_tok_abc123_xYz789...
```

**Token Validation:**
```typescript
async validate(token: string) {
  // 1. Split token parts
  const [prefix, tokenId, encryptedSign] = splitAccessToken(token);

  // 2. Decrypt signature
  const sign = decrypt(encryptedSign);

  // 3. Find token in database
  const accessToken = await prisma.accessToken.findUnique({
    where: { id: tokenId, sign }
  });

  // 4. Check expiration
  if (accessToken.expiredTime < new Date()) {
    throw new UnauthorizedException('Token expired');
  }

  // 5. Update last used time
  await prisma.accessToken.update({
    where: { id: tokenId },
    data: { lastUsedTime: new Date() }
  });

  return { userId: accessToken.userId, accessTokenId: tokenId };
}
```

### 3.4 Access Token Scopes

Tokens can be scoped to specific:
- **Actions**: `['base|read', 'record|read', 'record|create']`
- **Spaces**: `['sp_abc123']`
- **Bases**: `['bas_xyz789']`

```typescript
// Permission check with token
async getPermissions(resourceId, accessTokenId?) {
  const userPermissions = await this.getPermissionsByResourceId(resourceId);

  if (accessTokenId) {
    const token = await this.getAccessToken(accessTokenId);

    // Intersect user permissions with token scopes
    const tokenPermissions = token.scopes;
    return intersection(userPermissions, tokenPermissions);
  }

  return userPermissions;
}
```

---

## 4. OAuth Integration

### 4.1 Supported Providers

| Provider | Strategy | Configuration |
|----------|----------|---------------|
| **Google** | `passport-google-oauth20` | `BACKEND_GOOGLE_CLIENT_ID`, `BACKEND_GOOGLE_CLIENT_SECRET` |
| **GitHub** | `passport-github2` | `BACKEND_GITHUB_CLIENT_ID`, `BACKEND_GITHUB_CLIENT_SECRET` |
| **OIDC** | `passport-openidconnect` | `BACKEND_OIDC_*` configuration |

### 4.2 OAuth Flow

```
1. GET /api/auth/{provider}
   └─> Redirect to provider

2. Provider authenticates user
   └─> Redirect to callback

3. GET /api/auth/{provider}/callback
   ├─> Validate OAuth response
   ├─> Find or create user
   ├─> Create session
   └─> Redirect to application
```

### 4.3 Account Linking

```typescript
// Social login creates/links account
async findOrCreateUser(profile: IOAuthProfile) {
  // Check if account exists
  let account = await prisma.account.findUnique({
    where: {
      provider_providerId: {
        provider: profile.provider,
        providerId: profile.id
      }
    }
  });

  if (account) {
    return await prisma.user.findUnique({ where: { id: account.userId } });
  }

  // Create new user and account
  const user = await prisma.user.create({
    data: {
      name: profile.displayName,
      email: profile.email,
      avatar: profile.avatar
    }
  });

  await prisma.account.create({
    data: {
      userId: user.id,
      provider: profile.provider,
      providerId: profile.id,
      type: 'oauth'
    }
  });

  return user;
}
```

---

## 5. Authorization (RBAC)

### 5.1 Role Hierarchy

```
Owner (highest)
  └── Creator
        └── Editor
              └── Commenter
                    └── Viewer (lowest)
```

### 5.2 Role Capabilities

| Capability | Owner | Creator | Editor | Commenter | Viewer |
|------------|:-----:|:-------:|:------:|:---------:|:------:|
| Delete space/base | ✓ | | | | |
| Invite collaborators | ✓ | | | | |
| Manage permissions | ✓ | | | | |
| Create tables | ✓ | ✓ | | | |
| Delete tables | ✓ | ✓ | | | |
| Create/edit records | ✓ | ✓ | ✓ | | |
| Delete records | ✓ | ✓ | ✓ | | |
| Comment on records | ✓ | ✓ | ✓ | ✓ | |
| View records | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.3 Resource Hierarchy

```
Space (workspace level)
├── role: owner|creator|editor|commenter|viewer
│
└── Base (database level)
    ├── role: owner|creator|editor|commenter|viewer
    │   (inherits from space if not set)
    │
    └── Table/View/Record
        └── Inherits from base
```

---

## 6. Permission Model

### 6.1 Collaborator Model

```prisma
model Collaborator {
  id            String
  roleName      String    // 'owner' | 'creator' | 'editor' | 'commenter' | 'viewer'
  resourceId    String    // Space ID or Base ID
  resourceType  String    // 'space' | 'base'
  principalId   String    // User ID or Department ID
  principalType String    // 'user' | 'department'
  createdBy     String
  createdTime   DateTime
}
```

### 6.2 Permission Actions (70+)

**Space Actions:**
```typescript
'space|create'
'space|delete'
'space|read'
'space|update'
'space|invite_email'
'space|invite_link'
'space|grant_role'
```

**Base Actions:**
```typescript
'base|create'
'base|delete'
'base|read'
'base|update'
'base|read_all'
'base|invite_email'
'base|invite_link'
'base|table_import'
'base|table_export'
'base|authority_matrix_config'
'base|db_connection'
'base|query_data'
```

**Table/View/Field/Record Actions:**
```typescript
'table|create' | 'table|delete' | 'table|read' | 'table|update'
'view|create' | 'view|delete' | 'view|read' | 'view|update' | 'view|share'
'field|create' | 'field|delete' | 'field|read' | 'field|update'
'record|create' | 'record|delete' | 'record|read' | 'record|update' | 'record|comment' | 'record|copy'
```

### 6.3 Permission Resolution

```typescript
async getPermissionsByResourceId(resourceId: string) {
  // 1. Get collaborator record
  const collaborator = await this.getCollaborator(resourceId, userId);

  // 2. Determine role
  const role = collaborator?.roleName;

  // 3. If base, also check space-level role
  if (isBaseResource(resourceId)) {
    const base = await this.getBase(resourceId);
    const spaceRole = await this.getSpaceRole(base.spaceId, userId);

    // Use higher role
    role = getHigherRole(role, spaceRole);
  }

  // 4. Get permissions for role
  return getPermissions(role);
}
```

---

## 7. Guards & Decorators

### 7.1 AuthGuard

**Location**: `apps/nestjs-backend/src/features/auth/guard/auth.guard.ts`

```typescript
@Injectable()
export class AuthGuard extends PassportAuthGuard(['session', 'access-token', 'jwt']) {
  async canActivate(context: ExecutionContext) {
    // Check @Public() decorator
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) return true;

    try {
      return await this.validate(context);
    } catch (error) {
      // Check @EnsureLogin() decorator
      const ensureLogin = this.reflector.getAllAndOverride<boolean>(ENSURE_LOGIN);
      if (ensureLogin) {
        return response.redirect(`/auth/login?redirect=${encodeURIComponent(url)}`);
      }
      throw error;
    }
  }
}
```

### 7.2 PermissionGuard

**Location**: `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts`

```typescript
@Injectable()
export class PermissionGuard {
  async canActivate(context: ExecutionContext) {
    // 1. Check @Public()
    if (this.isPublic(context)) return true;

    // 2. Check @DisabledPermission()
    if (this.isDisabledPermission(context)) return true;

    // 3. Get required permissions from @Permissions()
    const requiredPermissions = this.getPermissions(context);

    // 4. Resolve resource ID from @ResourceMeta() or params
    const resourceId = this.resolveResourceId(context);

    // 5. Get user permissions
    const userPermissions = await this.permissionService.getPermissions(
      resourceId,
      accessTokenId
    );

    // 6. Check all required permissions are present
    return requiredPermissions.every(p => userPermissions.includes(p));
  }
}
```

### 7.3 Decorators

```typescript
// Skip authentication
@Public()

// Redirect to login on auth failure
@EnsureLogin()

// Require specific permissions
@Permissions('base|read', 'table|create')

// Allow access via API token
@TokenAccess()

// Skip permission checking
@DisabledPermission()

// Specify resource location
@ResourceMeta('baseId', 'params')  // Extract from route params
@ResourceMeta('spaceId', 'body')   // Extract from request body
@ResourceMeta('tableId', 'query')  // Extract from query string
```

### 7.4 Usage Examples

```typescript
@Controller('api/bases')
export class BaseController {
  // Public endpoint - no auth required
  @Public()
  @Get('public-info')
  getPublicInfo() { ... }

  // Requires authentication only
  @Get('my-bases')
  getMyBases() { ... }

  // Requires specific permission
  @Get(':baseId')
  @Permissions('base|read')
  @ResourceMeta('baseId', 'params')
  getBase(@Param('baseId') baseId: string) { ... }

  // Multiple permissions required
  @Post(':baseId/tables')
  @Permissions('base|read', 'table|create')
  @ResourceMeta('baseId', 'params')
  createTable(@Param('baseId') baseId: string) { ... }

  // Accessible via API token
  @TokenAccess()
  @Get(':baseId/records')
  @Permissions('record|read')
  getRecords() { ... }
}
```

---

## 8. Security Best Practices

### 8.1 Password Security

```typescript
// Bcrypt hashing
const saltRounds = 10;
const hash = await bcrypt.hash(password, saltRounds);

// Password comparison
const isValid = await bcrypt.compare(password, hash);
```

### 8.2 Rate Limiting

```typescript
// Login attempt tracking
const key = `signin:attempts:${email}`;
const attempts = await cache.get(key) || 0;

if (attempts >= MAX_LOGIN_ATTEMPTS) {
  throw new TooManyRequestsException('Account locked');
}

// Increment on failure
await cache.set(key, attempts + 1, LOCKOUT_MINUTES * 60);
```

### 8.3 Token Security

| Token Type | Encryption | Storage | Expiry |
|------------|------------|---------|--------|
| Session | Server-side | Redis/SQLite | 7 days |
| JWT | HS256 | Client | 20 days |
| Access Token | AES-128-CBC | Database | Configurable |
| OAuth State | None | Cache | 5 minutes |

### 8.4 Input Validation

```typescript
// Zod validation pipe
app.useGlobalPipes(new ValidationPipe({
  transform: true,
  stopAtFirstError: true,
  forbidUnknownValues: false
}));

// Request DTOs use Zod schemas
const CreateBaseSchema = z.object({
  name: z.string().min(1).max(255),
  spaceId: z.string().cuid()
});
```

### 8.5 SQL Injection Prevention

- **Prisma ORM**: Parameterized queries by default
- **Knex.js**: Query builder with parameterization
- **Raw queries**: Use `$queryRaw` with parameters

```typescript
// Safe: Parameterized
const users = await prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}`;

// Unsafe: String interpolation (NEVER do this)
// const users = await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE id = '${userId}'`);
```

### 8.6 CORS Configuration

```typescript
// Environment-based CORS
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(','),
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
});
```

### 8.7 Security Headers

```typescript
// Helmet middleware
app.use(helmet());

// Headers set:
// - X-Content-Type-Options: nosniff
// - X-Frame-Options: DENY
// - X-XSS-Protection: 1; mode=block
// - Strict-Transport-Security
```

---

## Key Configuration

### Environment Variables

```bash
# Authentication
BACKEND_JWT_SECRET=<32+ character secret>
BACKEND_SESSION_SECRET=<32+ character secret>
BACKEND_SESSION_EXPIRES_IN=7d

# Password Auth
SIGNIN_MAX_LOGIN_ATTEMPTS=5
SIGNIN_ACCOUNT_LOCKOUT_MINUTES=15

# OAuth
BACKEND_GOOGLE_CLIENT_ID=...
BACKEND_GOOGLE_CLIENT_SECRET=...
BACKEND_GITHUB_CLIENT_ID=...
BACKEND_GITHUB_CLIENT_SECRET=...

# Token Encryption
BACKEND_STORAGE_ENCRYPTION_ALGORITHM=aes-128-cbc
BACKEND_STORAGE_ENCRYPTION_KEY=<16 byte key>
BACKEND_STORAGE_ENCRYPTION_IV=<16 byte iv>
```

---

## Key Files Reference

| Component | Location |
|-----------|----------|
| Auth Module | `apps/nestjs-backend/src/features/auth/auth.module.ts` |
| Auth Guard | `apps/nestjs-backend/src/features/auth/guard/auth.guard.ts` |
| Permission Guard | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` |
| Permission Service | `apps/nestjs-backend/src/features/auth/permission.service.ts` |
| Session Store | `apps/nestjs-backend/src/features/auth/session/session-store.service.ts` |
| Access Token | `apps/nestjs-backend/src/features/access-token/` |
| Role Constants | `packages/core/src/auth/role/constant.ts` |
| Action Definitions | `packages/core/src/auth/actions.ts` |

---

*See also: [Main Architecture](../ANALYSIS.md) | [Database Architecture](./ARCHITECTURE-DATABASE.md)*
