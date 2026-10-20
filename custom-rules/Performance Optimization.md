# Performance Optimization Rules

## Overview

This document establishes mandatory rules and best practices for performance optimization in the MBG Apply application. Performance directly impacts user experience and SEO.

## Performance Budgets

### 1. Page Load Metrics

**Targets** (Mobile 4G):
- **Time to First Byte (TTFB)**: < 600ms
- **First Contentful Paint (FCP)**: < 1.8s
- **Largest Contentful Paint (LCP)**: < 2.5s
- **Time to Interactive (TTI)**: < 3.8s
- **Cumulative Layout Shift (CLS)**: < 0.1
- **First Input Delay (FID)**: < 100ms

### 2. Resource Budgets

**JavaScript**:
- Initial bundle: < 200KB (gzipped)
- Per route: < 100KB (gzipped)
- Total: < 500KB (gzipped)

**CSS**:
- Critical CSS: < 50KB
- Total CSS: < 100KB
- Per component: < 10KB

**Images**:
- Hero image: < 200KB
- Thumbnails: < 50KB
- Icons: Use SVG or inline

**Fonts**:
- Total fonts: < 100KB
- Subset fonts when possible
- Use system fonts as fallback

### 3. API Response Times

**Targets**:
- Simple queries: < 100ms
- Complex queries: < 500ms
- Database operations: < 300ms
- External API calls: < 1000ms

## Frontend Optimization

### 1. Code Splitting

**MUST** implement route-based code splitting:

```typescript
// Dynamic imports for large components
const AdminPanel = dynamic(() => import('@/components/AdminPanel'), {
  loading: () => <Skeleton />,
  ssr: false
})

// Lazy load heavy libraries
const Chart = dynamic(() => import('recharts'), {
  ssr: false
})
```

### 2. Image Optimization

**MUST** use Next.js Image component:

```typescript
import Image from 'next/image'

// ✅ GOOD - Optimized
<Image
  src="/hero.jpg"
  alt="Description"
  width={800}
  height={600}
  priority // For LCP image
  placeholder="blur"
/>

// ❌ BAD - Unoptimized
<img src="/hero.jpg" alt="Description" />
```

**Image Guidelines**:
- Use WebP format
- Provide width/height
- Use `priority` for above-fold images
- Use `loading="lazy"` for below-fold
- Compress images before upload
- Use CDN for image delivery

### 3. Font Optimization

**MUST** optimize web fonts:

```typescript
// next/font
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // Prevent invisible text
  preload: true
})

// ✅ GOOD - Subset fonts
font-family: Inter;
unicode-range: U+0000-00FF; // Latin only

// ❌ BAD - Load entire font
font-family: 'Noto Sans'; // All languages
```

**Font Loading Strategy**:
- Use `font-display: swap`
- Preload critical fonts
- Self-host fonts
- Use variable fonts
- Subset to required characters

### 4. CSS Optimization

**MUST** minimize CSS:

```typescript
// ✅ GOOD - Tailwind with purge
module.exports = {
  content: [
    './app/**/*.{js,ts,jsx,tsx}',
    './components/**/*.{js,ts,jsx,tsx}'
  ],
  // Removes unused CSS
}

// ✅ GOOD - Critical CSS inline
<style dangerouslySetInnerHTML={{
  __html: criticalCss
}} />

// ❌ BAD - Large CSS bundles
import 'bootstrap/dist/css/bootstrap.min.css'
```

**CSS Guidelines**:
- Use Tailwind CSS (utility-first)
- Purge unused styles
- Inline critical CSS
- Avoid `@import` in CSS
- Use CSS modules for components

### 5. JavaScript Optimization

**MUST** optimize JavaScript:

```typescript
// ✅ GOOD - Tree shaking
import { useState } from 'react'
import { format } from 'date-fns/format'

// ❌ BAD - Entire library imported
import * as React from 'react'
import dateFns from 'date-fns'

// ✅ GOOD - Lazy component
const HeavyComponent = lazy(() => import('./HeavyComponent'))

// ❌ BAD - Always loaded
import HeavyComponent from './HeavyComponent'
```

**JavaScript Guidelines**:
- Use tree shaking
- Minimize third-party libraries
- Remove console.log in production
- Use Web Workers for heavy computation
- Defer non-critical scripts

### 6. React Optimization

**MUST** optimize React components:

```typescript
// ✅ GOOD - Memoization
const MemoizedComponent = memo(Component)

const expensiveValue = useMemo(() => {
  return computeExpensiveValue(a, b)
}, [a, b])

const handleClick = useCallback(() => {
  doSomething(a, b)
}, [a, b])

// ❌ BAD - Unnecessary re-renders
const value = computeExpensiveValue(a, b) // Every render
const handleClick = () => doSomething() // New function every render
```

**React Guidelines**:
- Use `memo()` for expensive components
- Use `useMemo()` for expensive calculations
- Use `useCallback()` for event handlers
- Avoid inline functions in JSX
- Use keys properly in lists

## Backend Optimization

### 1. Database Query Optimization

**MUST** optimize database queries:

```typescript
// ✅ GOOD - Select only needed fields
const users = await prisma.user.findMany({
  select: {
    id: true,
    firstName: true,
    lastName: true
  }
})

// ❌ BAD - Select all fields
const users = await prisma.user.findMany()

// ✅ GOOD - Use indexes
await prisma.application.findMany({
  where: { status: 'SUBMITTED' } // Has index
})

// ❌ BAD - No index
await prisma.application.findMany({
  where: {
    personalStatement: { contains: 'keyword' } // No index
  }
})

// ✅ GOOD - Batch queries
const [users, apps] = await Promise.all([
  prisma.user.findMany(),
  prisma.application.findMany()
])

// ❌ BAD - Sequential queries
const users = await prisma.user.findMany()
const apps = await prisma.application.findMany()
```

**Database Guidelines**:
- Select only needed fields
- Use appropriate indexes
- Avoid N+1 queries
- Use pagination
- Batch database operations
- Use transactions sparingly

### 2. API Response Optimization

**MUST** optimize API responses:

```typescript
// ✅ GOOD - Cached response
return NextResponse.json(data, {
  headers: {
    'Cache-Control': 'private, max-age=60, stale-while-revalidate=300'
  }
})

// ✅ GOOD - Compressed response
import { gzip } from 'zlib'

const compressed = gzip(JSON.stringify(data))

// ✅ GOOD - Paginated response
const page = parseInt(searchParams.get('page') || '1')
const perPage = 50

const data = await prisma.application.findMany({
  take: perPage,
  skip: (page - 1) * perPage
})
```

**API Guidelines**:
- Enable compression
- Use caching headers
- Implement pagination
- Return only needed data
- Use ETags for conditional requests

### 3. Connection Pooling

**MUST** use connection pooling:

```typescript
// ✅ GOOD - Prisma handles pooling
const prisma = new PrismaClient()

// Configuration in DATABASE_URL
postgresql://user:pass@host:5432/db?connection_limit=10

// ❌ BAD - New connection each time
const prisma = new PrismaClient()
// ... close connection
await prisma.$disconnect()
```

### 4. Caching Strategy

**MUST** implement caching:

```typescript
// ✅ GOOD - Response caching
const cached = cache.get(key)
if (cached) return cached

const data = await fetchData()
cache.set(key, data, 300) // 5 minutes

// ✅ GOOD - Memoization
const memoizedFn = memoize(expensiveFn)

// Future: Redis caching
import Redis from 'ioredis'
const redis = new Redis(process.env.REDIS_URL)

const cached = await redis.get(key)
if (cached) return JSON.parse(cached)

const data = await fetchData()
await redis.setex(key, 300, JSON.stringify(data))
```

**Caching Guidelines**:
- Cache frequently accessed data
- Set appropriate TTL
- Invalidate on updates
- Use Redis for distributed caching
- Cache at multiple levels

## Network Optimization

### 1. HTTP/2 & HTTP/3

**MUST** leverage modern protocols:

- Use HTTP/2 (automatic on Vercel)
- Enable server push (cautiously)
- Use multiplexing
- Compress headers

### 2. CDN Usage

**MUST** use CDN for static assets:

```typescript
// Vercel CDN (automatic)
// Static files in /public automatically served via CDN

// External CDN for images
<Image
  src="https://cdn.example.com/image.jpg"
  loader={customLoader}
/>
```

### 3. Resource Hints

**MUST** use resource hints:

```typescript
// Preload critical resources
<link rel="preload" href="/critical.css" as="style" />
<link rel="preload" href="/hero.jpg" as="image" />

// Prefetch next page
<link rel="prefetch" href="/next-page" />

// DNS prefetch
<link rel="dns-prefetch" href="https://api.example.com" />

// Preconnect to origin
<link rel="preconnect" href="https://fonts.googleapis.com" />
```

### 4. Compression

**MUST** enable compression:

```typescript
// Vercel (automatic Brotli/Gzip)
// Configured in vercel.json or next.config

// API responses
return NextResponse.json(data, {
  headers: {
    'Content-Encoding': 'gzip'
  }
})
```

## Monitoring & Measurement

### 1. Performance Monitoring

**MUST** track performance metrics:

```typescript
// Web Vitals
import { getCLS, getFID, getLCP } from 'web-vitals'

getCLS(console.log)
getFID(console.log)
getLCP(console.log)

// Custom metrics
const startTime = performance.now()
await operation()
const duration = performance.now() - startTime

if (duration > 1000) {
  console.warn('Slow operation:', duration)
}
```

### 2. Lighthouse

**MUST** achieve minimum scores:

- **Performance**: > 90
- **Accessibility**: > 95
- **Best Practices**: > 95
- **SEO**: > 95

**Run regularly**:
```bash
# CLI
npx lighthouse https://apply.mbg.mn --view

# CI/CD
npm run lighthouse
```

### 3. Core Web Vitals

**MUST** pass Core Web Vitals:

- **LCP**: < 2.5s (75th percentile)
- **FID**: < 100ms (75th percentile)
- **CLS**: < 0.1 (75th percentile)

### 4. Real User Monitoring

**SHOULD** implement RUM:

```typescript
// Vercel Analytics (built-in)
import { Analytics } from '@vercel/analytics/react'

<Analytics />

// Custom events
if (typeof window !== 'undefined' && window.va) {
  window.va('track', 'slow_load', {
    page: window.location.pathname,
    duration
  })
}
```

## Common Performance Anti-Patterns

### ❌ Don'ts

```typescript
// ❌ BAD - Large bundle
import _ from 'lodash'

// ❌ BAD - Blocking render
const data = await fetch('/api/data')
return <Component data={data} />

// ❌ BAD - No memoization
const filtered = items.filter(expensive) // Every render

// ❌ BAD - Unnecessary re-renders
<Component onClick={() => handler()} />

// ❌ BAD - Large images
<img src="/10mb-image.jpg" />

// ❌ BAD - No lazy loading
import HeavyComponent from './Heavy'

// ❌ BAD - Inline styles
<div style={{ color: 'red', fontSize: 16 }} />

// ❌ BAD - N+1 queries
for (const user of users) {
  const apps = await prisma.application.findMany({
    where: { userId: user.id }
  })
}
```

### ✅ Do's

```typescript
// ✅ GOOD - Tree shaking
import { debounce } from 'lodash-es'

// ✅ GOOD - Non-blocking
const { data } = useSWR('/api/data')

// ✅ GOOD - Memoized
const filtered = useMemo(
  () => items.filter(expensive),
  [items]
)

// ✅ GOOD - Stable reference
const handleClick = useCallback(() => handler(), [])

// ✅ GOOD - Optimized image
<Image src="/image.jpg" width={800} height={600} />

// ✅ GOOD - Lazy loaded
const Heavy = dynamic(() => import('./Heavy'))

// ✅ GOOD - Tailwind classes
<div className="text-red-500 text-base" />

// ✅ GOOD - Single query
const usersWithApps = await prisma.user.findMany({
  include: { applications: true }
})
```

## Performance Checklist

### Before Deployment

- [ ] Lighthouse score > 90
- [ ] Core Web Vitals passing
- [ ] Bundle size within budget
- [ ] Images optimized
- [ ] Fonts optimized
- [ ] CSS purged
- [ ] JavaScript minified
- [ ] API responses cached
- [ ] Database queries optimized
- [ ] No console.log statements
- [ ] Source maps disabled (production)
- [ ] Compression enabled
- [ ] CDN configured
- [ ] Performance monitoring active

### Continuous Monitoring

- [ ] Track Web Vitals
- [ ] Monitor API response times
- [ ] Check database query performance
- [ ] Review bundle size changes
- [ ] Analyze user metrics
- [ ] Set up performance alerts
- [ ] Regular Lighthouse audits

## Tools & Resources

### Performance Testing

- **Lighthouse**: Automated audits
- **WebPageTest**: Detailed analysis
- **Chrome DevTools**: Profiling
- **Vercel Analytics**: Real user data

### Bundle Analysis

- **@next/bundle-analyzer**: Bundle visualization
- **source-map-explorer**: Bundle exploration
- **webpack-bundle-analyzer**: Webpack bundles

### Monitoring

- **Vercel Analytics**: Built-in monitoring
- **Google Analytics**: User metrics
- **Sentry**: Error tracking
- **Datadog**: APM (future)

## Resources

- [Web.dev Performance](https://web.dev/performance/)
- [Next.js Performance](https://nextjs.org/docs/advanced-features/measuring-performance)
- [Core Web Vitals](https://web.dev/vitals/)
- [Backend Development Rules](./BACKEND_DEVELOPMENT_RULES.md)

---

**Last Updated**: October 2025  
**Version**: 1.0  
**Status**: Mandatory for all performance-related work
