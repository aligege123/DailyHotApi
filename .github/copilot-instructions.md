# DailyHotApi - AI Coding Agent Instructions

## Project Overview

DailyHotApi is a hot trending data aggregation API service built with **Hono** framework and TypeScript. It aggregates real-time hot lists from 50+ Chinese platforms (Weibo, Zhihu, Bilibili, Douyin, etc.) and provides both JSON and RSS output formats.

**Tech Stack**: Hono (web framework), Node.js 20+, TypeScript, Axios, Cheerio (web scraping), Winston (logging), NodeCache/Redis (dual caching)

## Architecture

### Auto-Registry Pattern

All routes in `src/routes/*.ts` are **automatically discovered and registered** by `src/registry.ts` using filesystem scanning:

- Each route file exports a `handleRoute(c, noCache)` function
- Routes are mounted as `/{filename}` (e.g., `src/routes/zhihu.ts` → `/zhihu`)
- Query params: `?cache=false`, `?limit=10`, `?rss=true` are automatically handled
- No manual route registration needed - just create `src/routes/newsite.ts` and it's live

### Dual-Layer Caching System

1. **Primary**: NodeCache (in-memory, 100 keys max, 3600s TTL)
2. **Fallback**: Redis (optional, configured via env vars `REDIS_HOST`, `REDIS_PORT`, etc.)
3. Cache key = request URL. All `get()`/`post()` calls in `src/utils/getData.ts` auto-cache unless `noCache: true`

### Data Flow

```
Client Request → registry.ts → routes/{site}.ts → getData.ts (check cache) → External API → Transform data → Return RouterData
```

## Route Development Pattern

**Every route file must follow this structure:**

```typescript
// src/routes/example.ts
import type { RouterData } from "../types.js";
import { get } from "../utils/getData.js";
import { getTime } from "../utils/getTime.js";

export const handleRoute = async (c, noCache: boolean) => {
  const listData = await getList(noCache);
  const routeData: RouterData = {
    name: "example", // Must match filename
    title: "Example Site",
    type: "热榜",
    link: "https://example.com",
    total: listData.data?.length || 0,
    ...listData,
  };
  return routeData;
};

const getList = async (noCache: boolean) => {
  const url = "https://api.example.com/hot";
  const result = await get({ url, noCache });
  return {
    ...result,
    data: result.data.map((item) => ({
      id: item.id,
      title: item.title,
      desc: item.excerpt,
      cover: item.image,
      hot: item.score,
      timestamp: getTime(item.created_at), // Converts to Unix timestamp
      url: item.link,
      mobileUrl: item.mobile_link,
    })),
  };
};
```

**Key Rules:**

- Always export `handleRoute` as the entry point
- Use `getData.ts` helpers (`get`/`post`) for HTTP requests - they handle caching automatically
- Transform external API response to `ListItem[]` format (see `src/types.d.ts`)
- Handle special requirements via `src/config.ts` env vars (e.g., `config.ZHIHU_COOKIE`)

## Development Workflow

```bash
# Install (requires pnpm)
pnpm install

# Dev with hot reload & cache disabled
pnpm dev

# Dev with cache enabled (faster)
pnpm dev:cache

# Build
pnpm build

# Production
pnpm start
```

**Docker:**

```bash
docker build -t dailyhot-api .
docker run -p 6688:6688 -d dailyhot-api
```

## Configuration

All config via `.env` or environment variables (see `src/config.ts`):

- `PORT=6688` - Server port
- `CACHE_TTL=3600` - Cache expiration (seconds)
- `REQUEST_TIMEOUT=6000` - HTTP timeout (ms)
- `ALLOWED_DOMAIN=*` - CORS origins
- `REDIS_HOST/PORT/PASSWORD/DB` - Optional Redis config
- `ZHIHU_COOKIE` - Site-specific authentication (some platforms require cookies)
- `FILTER_WEIBO_ADVERTISEMENT=true` - Platform-specific filtering

## Special Patterns

### Token Extraction

Some sites require dynamic tokens (see `src/utils/getToken/*.ts`):

```typescript
// Example: src/routes/bilibili.ts
import { getBilibiliToken } from "../utils/getToken/bilibili.js";
const cookie = await getBilibiliToken();
const result = await get({ url, headers: { Cookie: cookie } });
```

### RSS Generation

Routes automatically support RSS via `?rss=true` or global `RSS_MODE=true`:

- `src/utils/getRSS.ts` converts `RouterData` to RSS XML
- Uses the `feed` package for RSS 2.0 format

### Logging

Use `src/utils/logger.ts` (Winston) - automatically logs to console and optionally `logs/*.log`:

```typescript
import logger from "./utils/logger.js";
logger.info("✅ Success");
logger.error("❌ Error details");
```

## Testing Routes

Access any route:

```
GET http://localhost:6688/{route-name}
GET http://localhost:6688/{route-name}?cache=false&limit=10&rss=true
GET http://localhost:6688/all  # Lists all available routes
```

## Common Pitfalls

1. **File extensions**: Always use `.js` in imports despite `.ts` source (ESM requirement)
2. **Cache persistence**: Remember cache lives in-memory by default - restarts clear it unless Redis configured
3. **Rate limiting**: Some platforms block frequent requests - rely on cache, don't set `noCache` in production
4. **CORS**: Middleware in `src/app.tsx` handles this - update `ALLOWED_DOMAIN` if needed

## Adding New Routes

1. Create `src/routes/newsite.ts` with `handleRoute` export
2. Define data transformation to `RouterData` format
3. Test at `http://localhost:6688/newsite`
4. No need to modify `registry.ts` - auto-discovered!

## Project Structure Logic

- `src/index.ts` - Entry point, starts Hono server
- `src/app.tsx` - Hono app setup (middleware, CORS, static files)
- `src/registry.ts` - Auto-discovers and mounts all routes
- `src/routes/*.ts` - Individual platform scrapers (50+ sites)
- `src/utils/*.ts` - Shared utilities (HTTP, cache, logging, time, RSS)
- `src/views/*.tsx` - JSX components for HTML pages (home, 404, error)
- `src/types.d.ts` - Core type definitions (`RouterData`, `ListItem`, etc.)
