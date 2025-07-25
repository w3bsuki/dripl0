# Supabase + SvelteKit Performance Best Practices Guide

## Table of Contents
1. [Supabase Query Optimization](#1-supabase-query-optimization)
2. [Row Level Security (RLS) Best Practices](#2-row-level-security-rls-best-practices)
3. [Connection Management & Pooling](#3-connection-management--pooling)
4. [Image Optimization Strategies](#4-image-optimization-strategies)
5. [SvelteKit Performance Optimization](#5-sveltekit-performance-optimization)
6. [Caching Strategies](#6-caching-strategies)
7. [Architecture Patterns](#7-architecture-patterns)
8. [Monitoring & Performance Analysis](#8-monitoring--performance-analysis)

---

## 1. Supabase Query Optimization

### Index Strategies

#### Use the Index Advisor
```sql
-- Automatically detect needed indexes
SELECT * FROM index_advisor('
  SELECT book.id 
  FROM book 
  WHERE title = $1
');
```

#### Choose the Right Index Type
```sql
-- B-tree (default) - for equality and range queries
CREATE INDEX idx_users_email ON users(email);

-- BRIN - for time-series data (10x smaller than B-tree)
CREATE INDEX idx_orders_created_at ON orders USING brin(created_at);

-- GIN - for arrays and JSONB
CREATE INDEX idx_user_tags ON users USING gin(tags);

-- Partial indexes - for frequently filtered subsets
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Composite indexes - for multi-column queries
CREATE INDEX idx_users_signup_priority ON users(sign_up_date, priority);

-- Expression indexes - for transformed data
CREATE INDEX idx_users_email_lower ON users(lower(email));
```

### Query Patterns

#### Optimize Joins
```sql
-- ✅ Good: Pre-filter with CTE
WITH active_users AS (
  SELECT id FROM users WHERE status = 'active'
)
SELECT * FROM orders o
JOIN active_users u ON u.id = o.user_id;

-- Always index foreign keys
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

#### Batch Operations
```sql
-- Batch updates with CTE
WITH updates AS (
  SELECT id, new_price 
  FROM (VALUES 
    (1, 19.99),
    (2, 29.99),
    (3, 39.99)
  ) AS t(id, new_price)
)
UPDATE products p
SET price = u.new_price
FROM updates u
WHERE p.id = u.id;
```

#### Use Materialized Views for Complex Aggregations
```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
  date_trunc('day', created_at) as sale_date,
  product_category,
  COUNT(*) as total_sales,
  SUM(amount) as revenue
FROM sales
GROUP BY 1, 2;

-- Index the materialized view
CREATE INDEX idx_sales_summary_date ON sales_summary(sale_date);

-- Refresh concurrently
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

---

## 2. Row Level Security (RLS) Best Practices

### Performance Optimization

#### Wrap Functions in SELECT
```sql
-- ✅ Fast: Function result is cached per statement
CREATE POLICY "user_policy" ON table_name
USING ((SELECT auth.uid()) = user_id);

-- ❌ Slow: Function called for each row
CREATE POLICY "user_policy" ON table_name
USING (auth.uid() = user_id);
```

#### Use Security Definer Functions
```sql
CREATE FUNCTION private.has_role(role_name text)
RETURNS boolean
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_id = (SELECT auth.uid()) 
    AND role = role_name
  );
END;
$$;
```

#### Minimize Joins in Policies
```sql
-- ✅ Good: No join to source table
CREATE POLICY "team_access" ON documents
USING (team_id IN (
  SELECT team_id FROM team_members 
  WHERE user_id = (SELECT auth.uid())
));
```

#### Always Specify Roles
```sql
CREATE POLICY "authenticated_only" ON table_name
TO authenticated  -- Prevents policy from running for anon users
USING ((SELECT auth.uid()) = user_id);
```

#### Index RLS Columns
Always create indexes on columns used in RLS policies for dramatic performance improvements.

---

## 3. Connection Management & Pooling

### Supavisor Configuration

#### Choose the Right Mode
- **Transaction Mode (port 6543)**: Best for serverless/edge functions
- **Session Mode (port 5432)**: For persistent connections needing session features
- **Direct Connection**: For persistent servers with IPv6 support

#### Connection Strings
```bash
# Transaction mode (serverless)
postgres://postgres.[PROJECT_REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres

# Session mode (persistent + IPv4)
postgres://postgres.[PROJECT_REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres

# Direct (IPv6 only)
postgresql://postgres:[PASSWORD]@db.[PROJECT_REF].supabase.co:5432/postgres
```

#### Pool Size Guidelines
- PostgREST API heavy usage: Keep pool size ≤40% of max connections
- Otherwise: Can allocate up to 80% to the pool
- Leave buffer for Auth server and utilities

#### Monitor Connections
```sql
-- Check active connections
SELECT 
  usename as role,
  application_name,
  state,
  COUNT(*) as connection_count
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY connection_count DESC;
```

---

## 4. Image Optimization Strategies

### Supabase Image Transformation

#### Basic Implementation
```javascript
// Automatic format optimization (WebP)
supabase.storage.from('bucket').getPublicUrl('image.jpg', {
  transform: {
    width: 500,
    height: 600,
    quality: 80, // 20-100, defaults to 80
    resize: 'contain' // 'contain', 'cover', or 'fill'
  },
})

// With signed URLs
supabase.storage.from('bucket').createSignedUrl('image.jpg', 60000, {
  transform: {
    width: 200,
    height: 200,
  },
})
```

#### Next.js Integration
```javascript
export default function supabaseLoader({ src, width, quality }) {
  return `https://${projectId}.supabase.co/storage/v1/render/image/public/${src}?width=${width}&quality=${quality || 75}`
}
```

### Modern Format Strategy (2024)

#### Format Priority
1. **AVIF** - 60-70% smaller than JPEG, full browser support
2. **WebP** - 25-35% smaller than JPEG
3. **JPEG** - Photograph fallback
4. **PNG** - Transparency fallback

#### Responsive Implementation
```html
<picture>
  <!-- AVIF variants -->
  <source 
    media="(max-width: 640px)" 
    srcset="img-640.avif 1x, img-1280.avif 2x" 
    type="image/avif">
  
  <!-- WebP variants -->
  <source 
    media="(max-width: 640px)" 
    srcset="img-640.webp 1x, img-1280.webp 2x" 
    type="image/webp">
  
  <!-- Fallback with responsive sizes -->
  <img 
    src="img-1200.jpg" 
    srcset="img-640.jpg 640w, img-1200.jpg 1200w"
    sizes="(max-width: 640px) 100vw, 50vw"
    alt="Responsive image"
    loading="lazy"
    decoding="async"
    width="1200"
    height="800">
</picture>
```

### CDN Strategy

#### Smart CDN Configuration
```javascript
// Control browser cache when uploading
const { data, error } = await supabase.storage
  .from('images')
  .upload('photo.jpg', file, {
    cacheControl: '3600', // 1 hour browser cache
    upsert: false
  });

// Cache busting with versioning
const imageUrl = `/storage/v1/object/sign/profile/avatar.jpg?version=${Date.now()}`;
```

---

## 5. SvelteKit Performance Optimization

### Core Strategies

#### Server-Side Setup
```javascript
// src/hooks.server.ts
import { createSupabaseServerClient } from '@supabase/auth-helpers-sveltekit'

export const handle = async ({ event, resolve }) => {
  event.locals.supabase = createSupabaseServerClient({
    supabaseUrl: PUBLIC_SUPABASE_URL,
    supabaseKey: PUBLIC_SUPABASE_ANON_KEY,
    event,
  })
  // Creates request-specific client for optimal performance
}
```

#### Avoid Request Waterfalls
```javascript
// ✅ Good: Server-side parallel loading
export async function load({ locals: { supabase } }) {
  const [user, items] = await Promise.all([
    supabase.auth.getUser(),
    supabase.from('items').select('*')
  ])
  return { user, items }
}
```

#### Streaming for Slow Data
```javascript
export async function load() {
  return {
    // Fast data loaded immediately
    posts: await getPosts(),
    // Slow data streamed later
    comments: getComments() // Returns promise, not awaited
  }
}
```

### Optimization Techniques

#### Image Optimization
- Use `@sveltejs/enhanced-img` for automatic processing
- Converts to modern formats (.avif, .webp)
- 50-70% smaller file sizes

#### Code Splitting
- Use dynamic imports `import(...)` for conditional code
- Svelte 5 can reduce bundle sizes by up to 50%
- Leverage Vite's automatic code splitting

#### Strategic Rendering Modes
```javascript
// Default SSR for SEO
export const ssr = true;

// CSR for interactive dashboards
export const ssr = false;

// Prerender static content
export const prerender = true;

// Smart detection
export const prerender = 'auto';
```

#### Preloading Strategy
```html
<!-- Enabled by default -->
<body data-sveltekit-preload-data="hover">
  <!-- Preloads on hover for instant navigation -->
</body>
```

---

## 6. Caching Strategies

### Database Level

#### Query Result Caching
- Postgres automatically caches in shared_buffers
- Monitor cache hit rates (should be >99%)
- Upgrade compute for more memory if needed

#### Check Cache Performance
```sql
-- Cache hit rates
SELECT 
  'index hit rate' as name,
  (sum(idx_blks_hit)) / nullif(sum(idx_blks_hit + idx_blks_read), 0) * 100 as ratio
FROM pg_statio_user_indexes
UNION ALL
SELECT 
  'table hit rate' as name,
  sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 as ratio
FROM pg_statio_user_tables;
```

### Application Level

#### HTTP Caching
```javascript
export async function load({ setHeaders }) {
  setHeaders({
    'cache-control': 'max-age=3600, s-maxage=86400',
    'cdn-cache-control': 'max-age=604800'
  })
}
```

#### Memoization
```javascript
import { memoize } from '$lib/utils'

const expensiveQuery = memoize(async (params) => {
  return await supabase
    .from('analytics')
    .select('*')
    .eq('user_id', params.userId)
})
```

### CDN Level

#### Supabase Storage CDN
- Automatic edge caching
- Smart invalidation on update/delete
- Works with signed URLs
- Configure with `cacheControl` on upload

---

## 7. Architecture Patterns

### Backend for Frontend (BFF)

#### Edge Functions for API Aggregation
```javascript
// Aggregate multiple queries in edge function
Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL'),
    Deno.env.get('SUPABASE_ANON_KEY')
  )

  // Parallel queries
  const [profile, posts, stats] = await Promise.all([
    supabase.from('profiles').select('*').single(),
    supabase.from('posts').select('*').limit(10),
    supabase.from('user_stats').select('*').single()
  ])

  return new Response(JSON.stringify({
    profile: profile.data,
    posts: posts.data,
    stats: stats.data
  }), {
    headers: { 'Content-Type': 'application/json' }
  })
})
```

### Realtime Optimization

```javascript
// Optimize subscriptions with filters
const subscription = supabase
  .channel('custom-filter-channel')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'messages',
      filter: 'channel_id=eq.123' // Filter at database level
    },
    (payload) => console.log(payload)
  )
  .subscribe()
```

### Static Generation Where Possible

```javascript
// Generate static pages for content
export async function load({ params }) {
  // This runs at build time when prerender = true
  const { data } = await supabase
    .from('articles')
    .select('*')
    .eq('slug', params.slug)
    .single()
  
  return { article: data }
}

export const prerender = true;
```

---

## 8. Monitoring & Performance Analysis

### Query Performance

#### Enable Query Analysis
```sql
-- Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries
SELECT 
  query,
  mean_exec_time,
  calls,
  total_exec_time
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY mean_exec_time DESC
LIMIT 10;
```

#### Use EXPLAIN ANALYZE
```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders 
WHERE created_at > '2024-01-01' 
ORDER BY total_amount DESC;
```

### Table Maintenance

```sql
-- Regular maintenance
VACUUM ANALYZE table_name;
REINDEX TABLE CONCURRENTLY table_name;

-- Check for bloat
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

### Core Web Vitals

#### Key Metrics to Monitor
- **FCP** (First Contentful Paint) < 1.8s
- **LCP** (Largest Contentful Paint) < 2.5s
- **TBT** (Total Blocking Time) < 200ms
- **CLS** (Cumulative Layout Shift) < 0.1
- **INP** (Interaction to Next Paint) < 200ms

#### Tools
- Lighthouse CI in build pipeline
- WebPageTest for real-world testing
- Supabase dashboard for database metrics
- Browser DevTools for client-side performance

---

## Implementation Checklist

### Database Optimization
- [ ] Run `index_advisor` on slow queries
- [ ] Create appropriate indexes (B-tree, BRIN, GIN)
- [ ] Optimize RLS policies with SELECT wrapping
- [ ] Index all RLS policy columns
- [ ] Set up materialized views for complex aggregations
- [ ] Configure connection pooling appropriately
- [ ] Enable `pg_stat_statements` for monitoring

### Frontend Optimization
- [ ] Implement server-side Supabase client
- [ ] Avoid request waterfalls with parallel loading
- [ ] Use streaming for slow data
- [ ] Enable prerendering where appropriate
- [ ] Implement proper image optimization
- [ ] Configure caching headers
- [ ] Use dynamic imports for code splitting

### Image Optimization
- [ ] Enable Supabase image transformations
- [ ] Implement responsive images with srcset
- [ ] Use modern formats (AVIF, WebP)
- [ ] Add lazy loading for below-fold images
- [ ] Configure CDN caching

### Monitoring
- [ ] Set up query performance monitoring
- [ ] Track Core Web Vitals
- [ ] Monitor cache hit rates
- [ ] Regular VACUUM and ANALYZE
- [ ] Track connection pool usage

---

## Performance Impact

Implementing these best practices can result in:
- **50-70%** reduction in image sizes with modern formats
- **50%** improvement in query performance with proper indexes
- **40-60%** faster page loads with proper caching
- **99%+** cache hit rates with optimized queries
- **2-3x** better performance for RLS policies
- **50%** smaller JavaScript bundles with Svelte 5

Remember: Always measure performance impacts in your specific use case and prioritize optimizations based on real user impact.