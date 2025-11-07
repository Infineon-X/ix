# Error Handling & Logging Rules

## Overview

This document establishes mandatory rules for error handling and logging in the MBG Apply application. Proper error handling and logging are critical for debugging, monitoring, and maintaining application health.

## Error Handling Rules

### 1. Error Handling Layers

**MANDATORY**: Implement error handling at all layers:

```typescript
// API Route Layer
export async function GET(request: NextRequest) {
  try {
    // Business logic
  } catch (error) {
    console.error('[API ERROR]', error)
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}

// Service Layer
export async function getUserData(userId: string) {
  try {
    return await prisma.user.findUnique({ where: { id: userId } })
  } catch (error) {
    console.error('[SERVICE ERROR] getUserData:', error)
    throw new Error('Failed to fetch user data')
  }
}

// Database Layer
// Prisma handles errors - catch at service layer
```

### 2. Error Response Format

**MANDATORY** standard error response:

```typescript
interface ErrorResponse {
  success: false
  error: string           // User-friendly message
  errorCode?: string      // Machine-readable code
  details?: any           // Development only
  timestamp?: string      // ISO timestamp
}

// Example
return NextResponse.json({
  success: false,
  error: 'Application not found',
  errorCode: 'NOT_FOUND',
  timestamp: new Date().toISOString()
}, { status: 404 })
```

### 3. Error Categories

**MUST** categorize errors appropriately:

#### Validation Errors (400)
```typescript
return NextResponse.json({
  success: false,
  error: 'Validation failed',
  errorCode: 'VALIDATION_ERROR',
  details: {
    fields: {
      email: 'Invalid email format',
      phone: 'Phone number required'
    }
  }
}, { status: 400 })
```

#### Authentication Errors (401)
```typescript
return NextResponse.json({
  success: false,
  error: 'Authentication required',
  errorCode: 'UNAUTHORIZED'
}, { status: 401 })
```

#### Authorization Errors (403)
```typescript
return NextResponse.json({
  success: false,
  error: 'Insufficient permissions',
  errorCode: 'FORBIDDEN'
}, { status: 403 })
```

#### Not Found Errors (404)
```typescript
return NextResponse.json({
  success: false,
  error: 'Resource not found',
  errorCode: 'NOT_FOUND'
}, { status: 404 })
```

#### Conflict Errors (409)
```typescript
return NextResponse.json({
  success: false,
  error: 'Resource already exists',
  errorCode: 'CONFLICT'
}, { status: 409 })
```

#### Server Errors (500)
```typescript
return NextResponse.json({
  success: false,
  error: 'Internal server error',
  errorCode: 'INTERNAL_ERROR'
}, { status: 500 })
```

### 4. Error Messages

**MUST** follow these rules:

✅ **DO**:
- Use clear, user-friendly messages
- Be specific about what went wrong
- Suggest solutions when possible
- Use consistent terminology

❌ **DON'T**:
- Expose internal implementation details
- Include stack traces in production
- Reveal database structure
- Expose file paths

```typescript
// GOOD
"Email address is already registered"
"Password must be at least 8 characters"
"Application not found"

// BAD
"Prisma error P2002: Unique constraint failed on users.email"
"Error in /app/api/users/route.ts line 42"
"SELECT * FROM users failed with code ECONNREFUSED"
```

### 5. Error Propagation

**MUST** handle errors at appropriate level:

```typescript
// Service Layer - Transform and throw
export async function createUser(data: UserData) {
  try {
    return await prisma.user.create({ data })
  } catch (error) {
    if (error.code === 'P2002') {
      throw new ConflictError('Email already exists')
    }
    throw new DatabaseError('Failed to create user')
  }
}

// API Layer - Catch and respond
export async function POST(request: NextRequest) {
  try {
    const user = await createUser(data)
    return NextResponse.json({ success: true, user })
  } catch (error) {
    if (error instanceof ConflictError) {
      return NextResponse.json(
        { success: false, error: error.message },
        { status: 409 }
      )
    }
    return NextResponse.json(
      { success: false, error: 'Failed to create user' },
      { status: 500 }
    )
  }
}
```

### 6. Async Error Handling

**MANDATORY** for promises:

```typescript
// GOOD - Proper async/await with try/catch
async function fetchData() {
  try {
    const data = await someAsyncOperation()
    return data
  } catch (error) {
    console.error('Fetch failed:', error)
    throw error
  }
}

// GOOD - Promise.all with error handling
try {
  const [users, apps] = await Promise.all([
    fetchUsers(),
    fetchApplications()
  ])
} catch (error) {
  console.error('Parallel fetch failed:', error)
}

// BAD - Unhandled promise rejection
someAsyncOperation() // No await, no catch
```

### 7. Database Error Handling

**MUST** handle Prisma errors:

```typescript
try {
  await prisma.user.create({ data })
} catch (error) {
  // Prisma error codes
  if (error.code === 'P2002') {
    // Unique constraint violation
    return { error: 'Email already exists' }
  }
  if (error.code === 'P2025') {
    // Record not found
    return { error: 'User not found' }
  }
  if (error.code === 'P2003') {
    // Foreign key constraint failed
    return { error: 'Invalid reference' }
  }
  
  // Unknown error
  console.error('Database error:', error)
  return { error: 'Database operation failed' }
}
```

### 8. Validation Error Handling

**MUST** validate inputs and return detailed errors:

```typescript
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email('Invalid email format'),
  firstName: z.string().min(1, 'First name required'),
  age: z.number().min(18, 'Must be at least 18 years old')
})

try {
  const validated = userSchema.parse(data)
} catch (error) {
  if (error instanceof z.ZodError) {
    return NextResponse.json({
      success: false,
      error: 'Validation failed',
      details: error.errors.map(e => ({
        field: e.path.join('.'),
        message: e.message
      }))
    }, { status: 400 })
  }
}
```

---

## Logging Rules

### 1. Log Levels

**MUST** use appropriate log levels:

```typescript
// ERROR - Critical issues requiring immediate attention
console.error('[ERROR]', 'Payment processing failed:', error)

// WARN - Potential issues that should be monitored
console.warn('[WARN]', 'API rate limit approaching threshold')

// INFO - Important business events
console.log('[INFO]', 'User registered:', userId)

// DEBUG - Detailed debugging information (development only)
if (process.env.NODE_ENV === 'development') {
  console.log('[DEBUG]', 'Request payload:', body)
}
```

### 2. Structured Logging

**MUST** use structured log format:

```typescript
// GOOD - Structured logging
console.log('[API]', {
  event: 'user_login',
  userId: 'user_123',
  timestamp: new Date().toISOString(),
  ipAddress: '192.168.1.1'
})

// BAD - Unstructured logging
console.log('User user_123 logged in from 192.168.1.1')
```

### 3. Log Context

**MUST** include context in logs:

```typescript
// Request context
console.log('[API]', {
  method: request.method,
  path: request.url,
  userId: session?.userId,
  duration: Date.now() - startTime
})

// Error context
console.error('[ERROR]', {
  error: error.message,
  stack: error.stack,
  context: {
    userId,
    applicationId,
    action: 'createApplication'
  }
})
```

### 4. Sensitive Data

**NEVER** log sensitive information:

❌ **DON'T LOG**:
- Passwords (plain or hashed)
- Session tokens
- API keys
- Credit card numbers
- Full email addresses
- Phone numbers
- Personal identification numbers

✅ **DO LOG** (masked):
```typescript
// GOOD - Masked sensitive data
console.log('[AUTH]', {
  email: email.replace(/(.{3}).*@/, '$1***@'),
  phone: phone.replace(/\d(?=\d{4})/g, '*')
})

// BAD - Exposing sensitive data
console.log('[AUTH]', {
  email: 'user@example.com',
  password: 'password123'
})
```

### 5. Production vs Development

**MUST** differentiate logging by environment:

```typescript
const isDevelopment = process.env.NODE_ENV === 'development'

// Development - verbose logging
if (isDevelopment) {
  console.log('[DEBUG] Request:', request)
  console.log('[DEBUG] Response:', response)
  console.log('[DEBUG] Query:', query)
}

// Production - essential logging only
console.log('[INFO] Request completed:', {
  path: request.url,
  duration: Date.now() - startTime,
  status: response.status
})
```

### 6. API Request Logging

**MUST** log API requests consistently:

```typescript
export async function GET(request: NextRequest) {
  const startTime = Date.now()
  const requestId = generateRequestId()
  
  console.log('[API] Request started:', {
    requestId,
    method: 'GET',
    path: request.url,
    timestamp: new Date().toISOString()
  })
  
  try {
    const result = await handleRequest()
    
    console.log('[API] Request completed:', {
      requestId,
      duration: Date.now() - startTime,
      status: 200
    })
    
    return NextResponse.json(result)
  } catch (error) {
    console.error('[API] Request failed:', {
      requestId,
      duration: Date.now() - startTime,
      error: error.message
    })
    
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

### 7. Database Query Logging

**SHOULD** log slow queries:

```typescript
const startTime = Date.now()

const users = await prisma.user.findMany()

const duration = Date.now() - startTime

if (duration > 1000) { // Slower than 1 second
  console.warn('[DB] Slow query detected:', {
    query: 'user.findMany',
    duration,
    threshold: 1000
  })
}
```

### 8. Business Event Logging

**MUST** log important business events:

```typescript
// User registration
console.log('[EVENT] User registered:', {
  userId,
  email: maskedEmail,
  timestamp: new Date().toISOString()
})

// Application submission
console.log('[EVENT] Application submitted:', {
  applicationId,
  userId,
  major,
  timestamp: new Date().toISOString()
})

// Status change
console.log('[EVENT] Application status changed:', {
  applicationId,
  oldStatus: 'SUBMITTED',
  newStatus: 'APPROVED',
  changedBy: adminId
})
```

### 9. Error Tracking

**MUST** log errors with full context:

```typescript
try {
  await criticalOperation()
} catch (error) {
  console.error('[ERROR] Critical operation failed:', {
    error: error.message,
    stack: error.stack,
    context: {
      userId,
      operation: 'criticalOperation',
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV
    }
  })
  
  // Send to error tracking service (future)
  // await reportError(error, context)
}
```

### 10. Audit Logging

**MUST** log admin actions for audit trail:

```typescript
import { createAuditLog } from '@/lib/audit'

await createAuditLog(session, {
  action: 'UPDATE',
  entityType: 'Application',
  entityId: applicationId,
  entityTitle: `${user.firstName} ${user.lastName}`,
  changes: {
    status: { from: 'SUBMITTED', to: 'APPROVED' }
  }
})
```

---

## Error Handling Patterns

### 1. Try-Catch Wrapper

```typescript
async function withErrorHandling<T>(
  operation: () => Promise<T>,
  context: string
): Promise<T> {
  try {
    return await operation()
  } catch (error) {
    console.error(`[ERROR] ${context}:`, error)
    throw error
  }
}

// Usage
const user = await withErrorHandling(
  () => prisma.user.create({ data }),
  'Creating user'
)
```

### 2. Error Boundary (Future)

```typescript
class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number = 500,
    public errorCode?: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

export function handleApiError(error: unknown) {
  if (error instanceof ApiError) {
    return NextResponse.json({
      success: false,
      error: error.message,
      errorCode: error.errorCode
    }, { status: error.statusCode })
  }
  
  console.error('[UNHANDLED ERROR]', error)
  return NextResponse.json({
    success: false,
    error: 'Internal server error'
  }, { status: 500 })
}
```

### 3. Retry Logic

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation()
    } catch (error) {
      lastError = error as Error
      console.warn(`[RETRY] Attempt ${i + 1}/${maxRetries} failed:`, error)
      
      if (i < maxRetries - 1) {
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
  }
  
  throw lastError!
}
```

---

## Monitoring & Alerting

### 1. Error Rate Monitoring

**SHOULD** track error rates:

```typescript
// Track errors per endpoint
const errorCounts = new Map<string, number>()

function trackError(endpoint: string) {
  const current = errorCounts.get(endpoint) || 0
  errorCounts.set(endpoint, current + 1)
  
  // Alert if error rate is high
  if (current > 10) {
    console.error('[ALERT] High error rate:', {
      endpoint,
      count: current
    })
  }
}
```

### 2. Performance Monitoring

**SHOULD** monitor response times:

```typescript
function trackPerformance(endpoint: string, duration: number) {
  console.log('[PERF]', {
    endpoint,
    duration,
    timestamp: new Date().toISOString()
  })
  
  if (duration > 3000) { // Slower than 3 seconds
    console.warn('[PERF] Slow response:', {
      endpoint,
      duration,
      threshold: 3000
    })
  }
}
```

---

## Best Practices Checklist

### Error Handling
- [ ] All async operations wrapped in try-catch
- [ ] Errors logged with context
- [ ] User-friendly error messages
- [ ] Appropriate HTTP status codes
- [ ] No sensitive data in errors
- [ ] Errors categorized correctly
- [ ] Database errors handled
- [ ] Validation errors detailed

### Logging
- [ ] Structured log format
- [ ] Appropriate log levels
- [ ] Sensitive data masked
- [ ] Request/response logged
- [ ] Errors logged with context
- [ ] Business events logged
- [ ] Slow queries logged
- [ ] Production vs development logging

### Monitoring
- [ ] Error rates tracked
- [ ] Performance monitored
- [ ] Alerts configured
- [ ] Audit logs created
- [ ] Health checks implemented

---

## Anti-Patterns to Avoid

### ❌ Silent Failures
```typescript
// BAD - Swallowing errors
try {
  await operation()
} catch (error) {
  // Nothing here!
}
```

### ❌ Generic Error Messages
```typescript
// BAD - Not helpful
return NextResponse.json({ error: 'Error occurred' })
```

### ❌ Logging Everything
```typescript
// BAD - Too verbose
console.log('Step 1')
console.log('Step 2')
console.log('Step 3')
```

### ❌ Exposing Internal Errors
```typescript
// BAD - Security risk
return NextResponse.json({
  error: error.stack // Never expose stack traces
})
```

---

## Enforcement

### Code Review Checklist
- [ ] All errors handled appropriately
- [ ] Logging follows standards
- [ ] No sensitive data logged
- [ ] Error messages user-friendly
- [ ] Context included in logs
- [ ] Appropriate log levels used

### Automated Checks
- [ ] Linter rules for error handling
- [ ] Tests for error cases
- [ ] Log format validation
- [ ] Sensitive data detection

---

## Resources

- [Error Handling Best Practices](https://www.joyent.com/node-js/production/design/errors)
- [Logging Best Practices](https://docs.datadoghq.com/logs/guide/log-collection-best-practices/)
- [Backend Development Rules](./BACKEND_DEVELOPMENT_RULES.md)
- [Security Best Practices](./SECURITY_BEST_PRACTICES.md)

---

**Last Updated**: October 2025  
**Version**: 1.0  
**Status**: Mandatory for all backend development
