# NextFaster - FAQ

## What exactly is Partial Prerendering?
Partial Prerendering (PPR) is a rendering technique used in this codebase to precompute the shells of pages, which are then served statically from the edge when deployed. Dynamic data (such as cart information) is then streamed in after the initial page load. This creates a hybrid rendering approach where static content is served immediately while dynamic content is fetched asynchronously.

In the codebase, PPR is enabled through the `experimental.ppr` flag in `next.config.mjs`. This allows Next.js to determine which parts of a page can be statically generated and which parts need dynamic data. When a user visits a page, they immediately see the static shell, providing better perceived performance, while dynamic components (like personalized cart information) are loaded separately.

## What is In-function Concurrency?
In-function Concurrency is a technique used in this codebase to reduce compute usage by over 50%. It allows multiple asynchronous operations to run in parallel within a function, rather than sequentially.

The codebase implements this using the Effect library, particularly in scripts like `scripts/makeImages.ts` and `scripts/genProducts.ts`. For example, when generating multiple product images or processing database operations, the code uses `Effect.all` with a `concurrency` parameter to limit the number of concurrent operations:

```typescript
yield* Effect.all(
  products.map((product) => Effect.gen(function* () {
    // Process each product
  })),
  { concurrency: 10 } // Run up to 10 operations concurrently
);
```

This approach significantly reduces overall processing time and compute resources, as it avoids the overhead of waiting for each operation to complete before starting the next one.

## What is the point of the "jose" package? What is it used for in this codebase?
The "jose" package (JavaScript Object Signing and Encryption) is used in this codebase for token-based authentication and session management. Specifically, it's used to:

1. Sign JWT (JSON Web Tokens) for user sessions
2. Verify incoming JWT tokens to authenticate users

In `src/lib/session.ts`, the jose package is used to implement:
- `signToken()`: Creates a signed JWT token containing user session data
- `verifyToken()`: Validates an incoming token to authenticate users

These functions are crucial for the secure authentication system, allowing the application to maintain stateless, secure user sessions.

## What is PPR? How is it enabled in this codebase?
PPR (Partial Prerendering) is an experimental Next.js feature that allows parts of a page to be statically generated while other parts are rendered dynamically. It's enabled in this codebase through the Next.js configuration file (`next.config.mjs`):

```javascript
experimental: {
  ppr: true,
  inlineCss: true,
  reactCompiler: true,
}
```

When enabled, Next.js automatically determines which parts of a page can be prerendered statically and which parts need dynamic data. The static parts are served immediately from the edge, while dynamic parts are streamed in, providing a fast initial page load while still supporting dynamic content.

## What is the React Compiler? What does it do? How is it used in this codebase?
The React Compiler (formerly known as React Forget) is an optimizing compiler for React components that automatically detects which parts of a component need to re-render when state changes. It eliminates the need for manual optimizations like `React.memo`, `useMemo`, and `useCallback` in most cases.

In this codebase, the React Compiler is:
1. Enabled through the `experimental.reactCompiler: true` setting in `next.config.mjs`
2. Supported by the `babel-plugin-react-compiler` dependency in `package.json`
3. Used with the canary version of React (`"react": "19.0.0-rc-cd22717c-20241013"`)

The React Compiler analyzes components at build time to generate more efficient code, reducing unnecessary re-renders and improving overall application performance without requiring developers to implement complex optimization strategies manually.

## How is caching set up in this codebase?
Caching in this codebase is implemented at multiple levels:

1. **Data Caching**: The codebase uses `unstable_cache` from Next.js, wrapped in React's `cache` function (in `src/lib/unstable-cache.ts`). This provides deduplication of requests and caches database query results.

2. **Image Caching**: In `next.config.mjs`, images are aggressively cached with a TTL of 1 year:
   ```javascript
   images: {
     minimumCacheTTL: 31536000, // 1 year in seconds
     // ...
   }
   ```

3. **Data Queries**: In `src/lib/queries.ts`, database queries are wrapped with `unstable_cache` with specific revalidation periods:
   ```typescript
   export const getProductDetails = unstable_cache(
     (productSlug: string) => db.query.products.findFirst({ /* ... */ }),
     ["product"],
     { revalidate: 60 * 60 * 2 } // two hours
   );
   ```

4. **Static Generation**: The use of Partial Prerendering (PPR) allows page shells to be cached and served from the edge.

This multi-layered caching approach significantly improves performance by reducing database queries, API calls, and computation.

## How is static generation implemented for performance?
Static generation for performance is implemented through several techniques:

1. **Partial Prerendering (PPR)**: As described earlier, PPR precomputes page shells that are served statically from the edge.

2. **Data Caching**: Database queries in `src/lib/queries.ts` use `unstable_cache` with appropriate revalidation periods to avoid unnecessary database calls.

3. **Image Optimization**: Images are aggressively cached with a 1-year TTL and optimized through Next.js's image optimization.

4. **Prefetching**: The codebase implements sophisticated prefetching strategies:
   - Links prefetch both page data and images on hover or when visible in the viewport
   - Product images are prefetched with low priority to avoid interfering with critical requests

By combining these techniques, the application achieves high performance metrics while maintaining a dynamic and responsive user experience.

## How are rate limits set up in this codebase?
Rate limiting in this codebase is implemented using Upstash Redis and the `@upstash/ratelimit` package. The system provides different rate limit rules for different types of authentication actions:

1. **IP-based Rate Limiting for Login Attempts**:
   In `src/lib/rate-limit.ts`, the codebase defines a sliding window rate limiter for authentication:
   ```typescript
   export const authRateLimit = new Ratelimit({
     redis,
     limiter: Ratelimit.slidingWindow(5, "15 m"),
     analytics: true,
     prefix: "ratelimit:auth",
   });
   ```
   This limits users to 5 login attempts per IP address within a 15-minute window.

2. **Stricter Rate Limiting for Signups**:
   ```typescript
   export const signUpRateLimit = new Ratelimit({
     redis,
     limiter: Ratelimit.slidingWindow(1, "15 m"),
     analytics: true,
     prefix: "ratelimit:signup",
   });
   ```
   This implements a stricter limit of only 1 signup attempt per IP address within a 15-minute window to prevent mass account creation.

The rate limiting is applied in the authentication actions (`src/app/(login)/actions.ts`):

For signups:
```typescript
const ip = (await headers()).get("x-real-ip") ?? "local";
const rl2 = await signUpRateLimit.limit(ip);
if (!rl2.success) {
  return {
    error: {
      code: "AUTH_ERROR",
      message: "Too many signups. Try again later",
    },
  };
}
```

For login attempts:
```typescript
const ip = (await headers()).get("x-real-ip") ?? "local";
const rl = await authRateLimit.limit(ip);
if (!rl.success) {
  return {
    error: {
      code: "AUTH_ERROR",
      message: "Too many attempts. Try again later",
    },
  };
}
```

This implementation helps protect the application from brute force attacks and prevents abuse of the signup system.

## How is Redis used in this codebase?
Redis is primarily used in this codebase through Vercel KV (a Redis-compatible service) for two main purposes:

1. **Rate Limiting**:
   The codebase uses Upstash Redis (`@upstash/redis` package) to implement rate limiting as described in the previous section. It tracks login attempts and signup requests by IP address, enforcing limits to prevent abuse.

   ```typescript
   const redis = new Redis({
     url: process.env.KV_REST_API_URL,
     token: process.env.KV_REST_API_TOKEN,
   });
   ```

   This Redis instance is then used by the rate limiters to track and enforce request limits.

2. **Caching Support**:
   While not explicitly shown in the code snippets provided, the Vercel KV (Redis) service can also be used as the backend for Next.js data caching mechanisms. The `unstable_cache` implementation in the codebase uses Next.js's built-in caching which can be configured to use Vercel KV as its storage mechanism.

The codebase connects to Redis using environment variables:
```typescript
if (!process.env.KV_REST_API_URL || !process.env.KV_REST_API_TOKEN) {
  throw new Error(
    "Please link a Vercel KV instance or populate `KV_REST_API_URL` and `KV_REST_API_TOKEN`"
  );
}
```

This ensures that the application has access to a Redis instance for its rate limiting and potentially other caching/persistence needs.
