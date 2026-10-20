# Server Actions Guide

## Overview

Server Actions in Next.js 15 provide a powerful way to perform server-side mutations directly from client components without creating API routes. This guide covers the implementation and best practices for Server Actions in the MBG Apply application.

## What are Server Actions?

Server Actions are asynchronous functions that run on the server but can be called directly from client components. They provide:
- **Type-safe** server mutations
- **Simplified data flow** without manual API routes
- **Progressive enhancement** for forms
- **Automatic revalidation** of cached data

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Client Component                        │
│  ┌────────────────────────────────────────────┐    │
│  │  const result = await serverAction(data)   │    │
│  └────────────────┬───────────────────────────┘    │
│                   │                                  │
└───────────────────┼──────────────────────────────────┘
                    │ RPC Call
                    ▼
┌─────────────────────────────────────────────────────┐
│              Server Action (app/actions/*)           │
│  ┌────────────────────────────────────────────┐    │
│  │  'use server'                              │    │
│  │  export async function serverAction() {    │    │
│  │    // Server-side logic                    │    │
│  │    // Database operations                  │    │
│  │    // Validation                           │    │
│  │    return result                           │    │
│  │  }                                         │    │
│  └────────────────┬───────────────────────────┘    │
│                   │                                  │
└───────────────────┼──────────────────────────────────┘
                    │
                    ▼
              ┌─────────────┐
              │   Database  │
              │  (Prisma)   │
              └─────────────┘
```

## File Structure

```
app/
├── actions/
│   ├── application-actions.ts  # Application-related actions
│   ├── user-actions.ts         # User-related actions
│   ├── config.ts               # Configuration actions
│   └── [domain]-actions.ts     # Domain-specific actions
```

## Basic Server Action

### 1. Creating a Server Action

**File**: `app/actions/user-actions.ts`

```typescript
'use server'

import { auth, currentUser } from '@clerk/nextjs/server'
import { prisma } from '@/lib/prisma'

export async function createOrUpdateUserAction() {
  // 1. Authentication check
  const clerkUser = await currentUser()
  
  if (!clerkUser) {
    return { success: false, error: 'Not authenticated' }
  }
  
  // 2. Data extraction
  const email = clerkUser.emailAddresses[0]?.emailAddress
  
  if (!email) {
    return { success: false, error: 'No email found' }
  }
  
  // 3. Database operation
  try {
    const user = await prisma.user.upsert({
      where: { id: clerkUser.id },
      update: {
        firstName: clerkUser.firstName || '',
        lastName: clerkUser.lastName || '',
        avatarUrl: clerkUser.imageUrl,
        clerkEmail: email
      },
      create: {
        id: clerkUser.id,
        clerkEmail: email,
        firstName: clerkUser.firstName || '',
        lastName: clerkUser.lastName || '',
        avatarUrl: clerkUser.imageUrl
      }
    })
    
    // 4. Return result
    return { success: true, user }
  } catch (error) {
    console.error('User creation/update error:', error)
    return { success: false, error: 'Database operation failed' }
  }
}
```

### 2. Using in Client Component

```typescript
'use client'

import { createOrUpdateUserAction } from '@/app/actions/user-actions'
import { useEffect, useState } from 'react'

export function UserSync() {
  const [status, setStatus] = useState<string>('idle')
  
  useEffect(() => {
    async function syncUser() {
      setStatus('syncing')
      
      const result = await createOrUpdateUserAction()
      
      if (result.success) {
        setStatus('synced')
      } else {
        setStatus('error')
        console.error(result.error)
      }
    }
    
    syncUser()
  }, [])
  
  return <div>Status: {status}</div>
}
```

## Application Actions

### Create Application

**File**: `app/actions/application-actions.ts`

```typescript
'use server'

import { auth } from '@clerk/nextjs/server'
import { prisma } from '@/lib/prisma'
import { ApplicationStatus } from '@prisma/client'
import { revalidatePath } from 'next/cache'

interface ApplicationInput {
  firstName: string
  lastName: string
  email: string
  phone: string
  dateOfBirth: string
  studyPlan1: string
  studyLevel: string
  // ... other fields
}

export async function createApplicationAction(data: ApplicationInput) {
  // 1. Authenticate
  const { userId } = await auth()
  
  if (!userId) {
    return {
      success: false,
      error: 'Authentication required',
      status: 401
    }
  }
  
  // 2. Validate input
  const requiredFields = ['firstName', 'lastName', 'email', 'phone']
  for (const field of requiredFields) {
    if (!data[field]) {
      return {
        success: false,
        error: `Missing required field: ${field}`,
        status: 400
      }
    }
  }
  
  try {
    // 3. Check for existing application
    const existingApplication = await prisma.application.findFirst({
      where: { userId }
    })
    
    if (existingApplication) {
      return {
        success: false,
        error: 'Application already exists',
        errorCode: 'DUPLICATE_APPLICATION',
        status: 409
      }
    }
    
    // 4. Create application in transaction
    const application = await prisma.$transaction(async (tx) => {
      // Update user info
      await tx.user.update({
        where: { id: userId },
        data: {
          firstName: data.firstName,
          lastName: data.lastName,
          inputEmail: data.email,
          phone: data.phone,
          dateOfBirth: new Date(data.dateOfBirth)
        }
      })
      
      // Create application
      const app = await tx.application.create({
        data: {
          userId,
          major: data.studyPlan1,
          currentEducationLevel: data.studyLevel,
          status: ApplicationStatus.SUBMITTED,
          submittedAt: new Date(),
          // ... map other fields
        }
      })
      
      // Create default timeline steps
      const timelineSteps = [
        { stepNumber: 1, title: 'Application Submitted', status: 'COMPLETED' },
        { stepNumber: 2, title: 'Document Review', status: 'CURRENT' },
        { stepNumber: 3, title: 'Interview', status: 'UPCOMING' },
        { stepNumber: 4, title: 'Decision', status: 'UPCOMING' },
      ]
      
      await Promise.all(
        timelineSteps.map(step =>
          tx.timelineStep.create({
            data: {
              applicationId: app.id,
              ...step,
              estimate: '1-2 weeks',
              actionText: 'View'
            }
          })
        )
      )
      
      return app
    })
    
    // 5. Revalidate cache
    revalidatePath('/dashboard')
    
    // 6. Return success
    return {
      success: true,
      applicationId: application.id
    }
    
  } catch (error) {
    console.error('Application creation error:', error)
    return {
      success: false,
      error: 'Failed to create application',
      status: 500
    }
  }
}
```

### Update Application

```typescript
'use server'

export async function updateApplicationAction(
  applicationId: string,
  updates: Partial<ApplicationInput>
) {
  const { userId } = await auth()
  
  if (!userId) {
    return { success: false, error: 'Unauthorized' }
  }
  
  try {
    // Verify ownership
    const application = await prisma.application.findUnique({
      where: { id: applicationId },
      select: { userId: true, status: true }
    })
    
    if (!application) {
      return { success: false, error: 'Application not found' }
    }
    
    if (application.userId !== userId) {
      return { success: false, error: 'Forbidden' }
    }
    
    // Don't allow editing submitted applications
    if (application.status !== 'DRAFT') {
      return { success: false, error: 'Cannot edit submitted application' }
    }
    
    // Update
    await prisma.application.update({
      where: { id: applicationId },
      data: updates
    })
    
    revalidatePath(`/application/${applicationId}`)
    
    return { success: true }
  } catch (error) {
    console.error('Application update error:', error)
    return { success: false, error: 'Update failed' }
  }
}
```

## Form Integration

### Progressive Enhancement

Server Actions work without JavaScript:

```typescript
export function ApplicationForm() {
  return (
    <form action={createApplicationAction}>
      <input name="firstName" required />
      <input name="lastName" required />
      <input name="email" type="email" required />
      <button type="submit">Submit</button>
    </form>
  )
}
```

### With React Hook Form

```typescript
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { createApplicationAction } from '@/app/actions/application-actions'

const schema = z.object({
  firstName: z.string().min(1),
  lastName: z.string().min(1),
  email: z.string().email(),
  phone: z.string().min(8)
})

type FormData = z.infer<typeof schema>

export function ApplicationForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema)
  })
  
  const onSubmit = async (data: FormData) => {
    const result = await createApplicationAction(data)
    
    if (result.success) {
      // Redirect or show success
      window.location.href = `/application/${result.applicationId}`
    } else {
      // Show error
      alert(result.error)
    }
  }
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName')} />
      {errors.firstName && <span>{errors.firstName.message}</span>}
      
      <input {...register('lastName')} />
      {errors.lastName && <span>{errors.lastName.message}</span>}
      
      <input {...register('email')} type="email" />
      {errors.email && <span>{errors.email.message}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  )
}
```

## Error Handling

### Consistent Error Format

```typescript
'use server'

type ActionResult<T> = 
  | { success: true; data: T }
  | { success: false; error: string; errorCode?: string }

export async function myAction(): Promise<ActionResult<User>> {
  try {
    const user = await performOperation()
    return { success: true, data: user }
  } catch (error) {
    return {
      success: false,
      error: 'Operation failed',
      errorCode: 'OPERATION_ERROR'
    }
  }
}
```

### Client Error Handling

```typescript
const result = await myAction()

if (!result.success) {
  // Handle error
  console.error(result.error)
  
  if (result.errorCode === 'DUPLICATE_APPLICATION') {
    // Show specific message
  }
  
  return
}

// Use data
const user = result.data
```

## Validation

### Server-Side Validation with Zod

```typescript
'use server'

import { z } from 'zod'

const applicationSchema = z.object({
  firstName: z.string().min(1, 'First name required'),
  lastName: z.string().min(1, 'Last name required'),
  email: z.string().email('Invalid email'),
  phone: z.string().regex(/^\+?[1-9]\d{7,14}$/, 'Invalid phone'),
  dateOfBirth: z.string().refine(
    (date) => {
      const age = new Date().getFullYear() - new Date(date).getFullYear()
      return age >= 16 && age <= 100
    },
    'Must be between 16 and 100 years old'
  )
})

export async function createApplicationAction(data: unknown) {
  // Validate
  const validated = applicationSchema.safeParse(data)
  
  if (!validated.success) {
    return {
      success: false,
      error: 'Validation failed',
      errors: validated.error.flatten().fieldErrors
    }
  }
  
  // Use validated data
  const { firstName, lastName, email } = validated.data
  
  // Continue with database operation
}
```

## Revalidation & Caching

### Revalidate Paths

```typescript
'use server'

import { revalidatePath } from 'next/cache'

export async function updateUserAction(userId: string, data: UserData) {
  await prisma.user.update({
    where: { id: userId },
    data
  })
  
  // Revalidate specific paths
  revalidatePath('/dashboard')
  revalidatePath(`/profile/${userId}`)
}
```

### Revalidate Tags

```typescript
'use server'

import { revalidateTag } from 'next/cache'

export async function updateApplicationAction(applicationId: string) {
  await prisma.application.update({ ... })
  
  // Revalidate by tag
  revalidateTag('applications')
  revalidateTag(`application-${applicationId}`)
}
```

## Best Practices

### 1. Always Use 'use server'

```typescript
// At the top of file for all functions
'use server'

export async function action1() { }
export async function action2() { }

// Or per function
export async function action3() {
  'use server'
  // ...
}
```

### 2. Return Consistent Results

```typescript
// GOOD - Consistent structure
type Result<T> = { success: true; data: T } | { success: false; error: string }

export async function myAction(): Promise<Result<User>> {
  try {
    const user = await operation()
    return { success: true, data: user }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// BAD - Inconsistent
export async function badAction() {
  try {
    return await operation() // Sometimes returns data
  } catch (error) {
    throw error // Sometimes throws
  }
}
```

### 3. Validate Input

```typescript
// GOOD - Validate all inputs
export async function createUser(data: unknown) {
  const validated = userSchema.parse(data)
  // Use validated.field
}

// BAD - Trust input
export async function createUser(data: any) {
  await prisma.user.create({ data }) // Dangerous
}
```

### 4. Handle Authentication

```typescript
// GOOD - Check auth first
export async function protectedAction() {
  const { userId } = await auth()
  if (!userId) {
    return { success: false, error: 'Unauthorized' }
  }
  // Continue
}

// BAD - No auth check
export async function unprotectedAction() {
  // Anyone can call this
}
```

### 5. Use Transactions

```typescript
// GOOD - Atomic operations
export async function multiStepAction() {
  await prisma.$transaction(async (tx) => {
    await tx.user.create({ ... })
    await tx.application.create({ ... })
  })
}

// BAD - Not atomic
export async function nonAtomicAction() {
  await prisma.user.create({ ... })
  await prisma.application.create({ ... }) // May fail, leaving inconsistent state
}
```

### 6. Log Errors Appropriately

```typescript
// GOOD - Log with context
export async function myAction() {
  try {
    await operation()
  } catch (error) {
    console.error('Operation failed:', {
      action: 'myAction',
      timestamp: new Date(),
      error: error instanceof Error ? error.message : String(error)
    })
    return { success: false, error: 'Operation failed' }
  }
}
```

### 7. Revalidate After Mutations

```typescript
// GOOD - Revalidate affected paths
export async function updateData() {
  await prisma.data.update({ ... })
  revalidatePath('/data')
}

// BAD - No revalidation, stale cache
export async function updateDataNoRevalidate() {
  await prisma.data.update({ ... })
  // Cache still shows old data
}
```

## Testing Server Actions

### Unit Testing

```typescript
import { createApplicationAction } from '@/app/actions/application-actions'
import { auth } from '@clerk/nextjs/server'

jest.mock('@clerk/nextjs/server')
jest.mock('@/lib/prisma', () => ({
  prisma: {
    application: {
      create: jest.fn(),
      findFirst: jest.fn()
    }
  }
}))

describe('createApplicationAction', () => {
  it('creates application successfully', async () => {
    // Mock auth
    (auth as jest.Mock).mockResolvedValue({ userId: 'user_123' })
    
    // Mock no existing application
    (prisma.application.findFirst as jest.Mock).mockResolvedValue(null)
    
    // Mock create
    (prisma.application.create as jest.Mock).mockResolvedValue({
      id: 'app_123',
      userId: 'user_123'
    })
    
    // Test
    const result = await createApplicationAction(mockData)
    
    expect(result.success).toBe(true)
    expect(result.applicationId).toBe('app_123')
  })
  
  it('rejects unauthenticated request', async () => {
    (auth as jest.Mock).mockResolvedValue({ userId: null })
    
    const result = await createApplicationAction(mockData)
    
    expect(result.success).toBe(false)
    expect(result.error).toContain('Authentication')
  })
})
```

## Performance Considerations

### 1. Avoid Over-fetching

```typescript
// GOOD - Select only needed fields
export async function getUserName(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: { firstName: true, lastName: true }
  })
  return user
}

// BAD - Fetch everything
export async function getUserNameBad(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId }
  })
  return user // Returns all fields
}
```

### 2. Parallel Operations

```typescript
// GOOD - Parallel execution
export async function getDashboardData(userId: string) {
  const [user, applications, documents] = await Promise.all([
    prisma.user.findUnique({ where: { id: userId } }),
    prisma.application.findMany({ where: { userId } }),
    prisma.document.findMany({ where: { userId } })
  ])
  return { user, applications, documents }
}

// BAD - Sequential
export async function getDashboardDataSlow(userId: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } })
  const applications = await prisma.application.findMany({ where: { userId } })
  const documents = await prisma.document.findMany({ where: { userId } })
  return { user, applications, documents }
}
```

## Security Checklist

- [ ] Validate all inputs
- [ ] Check authentication
- [ ] Verify authorization
- [ ] Use transactions for multi-step operations
- [ ] Never expose sensitive data in errors
- [ ] Log security events
- [ ] Sanitize user input
- [ ] Rate limit expensive operations
- [ ] Implement proper error handling

## Common Pitfalls

### 1. Forgetting 'use server'
```typescript
// Will cause errors
export async function myAction() {
  // This runs on client!
}

// Correct
'use server'
export async function myAction() {
  // Runs on server
}
```

### 2. Not Handling Errors
```typescript
// BAD - Unhandled errors
export async function action() {
  await riskyOperation() // May throw
}

// GOOD - Handled errors
export async function action() {
  try {
    await riskyOperation()
    return { success: true }
  } catch (error) {
    return { success: false, error: 'Failed' }
  }
}
```

### 3. Returning Non-Serializable Data
```typescript
// BAD - Returns Date object
export async function getUser() {
  return await prisma.user.findUnique({ ... })
  // Date fields not serializable
}

// GOOD - Convert dates to strings
export async function getUser() {
  const user = await prisma.user.findUnique({ ... })
  return {
    ...user,
    createdAt: user.createdAt.toISOString()
  }
}
```

## Migration from API Routes

### Before (API Route)
```typescript
// app/api/create-app/route.ts
export async function POST(request: Request) {
  const body = await request.json()
  const result = await createApplication(body)
  return NextResponse.json(result)
}

// Client
const response = await fetch('/api/create-app', {
  method: 'POST',
  body: JSON.stringify(data)
})
const result = await response.json()
```

### After (Server Action)
```typescript
// app/actions/application-actions.ts
'use server'
export async function createApplicationAction(data) {
  const result = await createApplication(data)
  return result
}

// Client
const result = await createApplicationAction(data)
```

## Conclusion

Server Actions provide a powerful, type-safe way to handle server-side mutations in Next.js. They simplify the development process by:
- Eliminating boilerplate API routes
- Providing automatic type safety
- Enabling progressive enhancement
- Simplifying error handling
- Improving developer experience

For more information:
- [Next.js Server Actions Docs](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [Backend Development Rules](../rules/BACKEND_DEVELOPMENT_RULES.md)
- [API Routes Documentation](./API_ROUTES.md)
