# Search Tools Conversion Plan

## Decision: Option C (Hybrid Approach)

**Rationale:** Implement both Google News RSS and Anthropic Web Search tools to provide complementary capabilities for financial research.

### Why Hybrid?

1. **Different Use Cases:**
   - **Google News RSS**: Structured news search with reliable publishing dates, source attribution, and specific financial news focus
   - **Anthropic Web Search**: General web search with broader coverage, real-time information, and natural language understanding

2. **Financial Research Requirements:**
   - Financial analysts need **dated sources** (Google News provides reliable publication dates)
   - Need **diverse sources** (Web Search provides broader coverage)
   - Need **news-specific results** (Google News is specialized for news)
   - Need **real-time market data** (Web Search can access current information)

3. **Redundancy Benefits:**
   - Fallback if one service fails
   - Cross-validation of information
   - Different strengths for different query types

## Implementation Architecture

### Tool Structure

```
src/tools/search/
├── index.ts                 # Export all search tools
├── google-news.ts           # Google News RSS implementation
├── web-search.ts            # Anthropic Web Search wrapper
├── models.ts                # Shared TypeScript types
└── utils.ts                 # Shared utilities (RSS parsing, date parsing, etc.)
```

### Type Definitions

```typescript
// models.ts
export interface SearchResult {
  title: string;
  url: string;
  publishedDate: Date | null;
  source?: string;
}

export interface SearchToolInput {
  query: string;
  maxResults?: number;
}

export interface GoogleNewsSearchInput extends SearchToolInput {
  language?: string;
  country?: string;
}

export interface WebSearchInput extends SearchToolInput {
  allowedDomains?: string[];
  blockedDomains?: string[];
}
```

## Implementation Details

### 1. Google News RSS Search (TypeScript Port)

**Dependencies:**
- `rss-parser`: RSS/XML parsing (well-maintained, TypeScript support)
- `node-fetch` or `axios`: HTTP requests
- Built-in URL decoding (no TypeScript equivalent of googlenewsdecoder)

**Key Implementation Points:**

#### URL Decoding Strategy
Since there's no direct TypeScript port of `googlenewsdecoder`, we have three options:

**Option A: Direct HTTP Resolution (Recommended)**
```typescript
async function resolveGoogleNewsUrl(url: string): Promise<string> {
  if (!url || !url.includes('news.google.com')) {
    return url;
  }

  try {
    const response = await fetch(url, {
      method: 'HEAD',
      redirect: 'follow',
      timeout: 5000,
    });
    return response.url;
  } catch {
    return url;
  }
}
```

**Option B: Base64 Decoding (Partial)**
Many Google News URLs use base64 encoding. We can decode the URL parameter:
```typescript
function decodeGoogleNewsUrl(url: string): string {
  try {
    const urlObj = new URL(url);
    const articleParam = urlObj.searchParams.get('url');
    if (articleParam) {
      return Buffer.from(articleParam, 'base64').toString('utf-8');
    }
    return url;
  } catch {
    return url;
  }
}
```

**Option C: Use google-news-scraper Package**
The `google-news-scraper` npm package includes URL decoding functionality and TypeScript definitions.

**Recommendation:** Use Option A (HTTP redirect following) as the primary method with Option B as fallback.

#### RSS Parsing

```typescript
import Parser from 'rss-parser';

interface RSSItem {
  title?: string;
  link?: string;
  pubDate?: string;
  isoDate?: string;
}

async function parseGoogleNewsRSS(
  query: string,
  maxResults: number
): Promise<SearchResult[]> {
  const searchUrl = `https://news.google.com/rss/search?q=${encodeURIComponent(query)}&hl=en-US&gl=US&ceid=US:en`;

  const parser = new Parser<{}, RSSItem>();
  const feed = await parser.parseURL(searchUrl);

  const results: SearchResult[] = feed.items
    .slice(0, maxResults * 2) // Get extra in case some fail
    .map(item => ({
      title: cleanText(item.title || 'No title'),
      url: item.link || '',
      publishedDate: parseDate(item.pubDate || item.isoDate),
    }));

  // Resolve URLs in parallel
  const resolved = await Promise.all(
    results.map(async (result) => ({
      ...result,
      url: await resolveGoogleNewsUrl(result.url),
    }))
  );

  return resolved.slice(0, maxResults);
}
```

#### Text Cleaning Utilities

```typescript
function cleanText(text: string): string {
  if (!text) return text;

  // Remove HTML tags
  text = text.replace(/<[^>]+>/g, '');

  // Decode HTML entities
  text = decodeHTMLEntities(text);

  // Replace Unicode characters
  const replacements: Record<string, string> = {
    '\u2018': "'",
    '\u2019': "'",
    '\u201c': '"',
    '\u201d': '"',
    '\u2013': '-',
    '\u2014': '-',
    '\u2026': '...',
    '\u00a0': ' ',
    '\u00ae': '(R)',
    '\u2122': '(TM)',
  };

  for (const [unicode, replacement] of Object.entries(replacements)) {
    text = text.replace(new RegExp(unicode, 'g'), replacement);
  }

  // Normalize whitespace
  text = text.replace(/\s+/g, ' ').trim();

  return text;
}

function decodeHTMLEntities(text: string): string {
  const entities: Record<string, string> = {
    '&amp;': '&',
    '&lt;': '<',
    '&gt;': '>',
    '&quot;': '"',
    '&#39;': "'",
    '&apos;': "'",
  };

  return text.replace(/&[^;]+;/g, entity => entities[entity] || entity);
}
```

#### Date Parsing

```typescript
function parseDate(dateStr: string | undefined): Date | null {
  if (!dateStr) return null;

  try {
    // Try ISO date first (from rss-parser)
    const date = new Date(dateStr);
    if (!isNaN(date.getTime())) {
      return date;
    }
  } catch {}

  // Try RFC 822 format (RSS standard)
  try {
    const cleaned = dateStr.replace(/ GMT| \+0000/g, '');
    const date = new Date(cleaned);
    if (!isNaN(date.getTime())) {
      return date;
    }
  } catch {}

  // Fallback patterns
  const patterns = [
    /(\d{4}-\d{2}-\d{2})/,
    /(\d{1,2}\/\d{1,2}\/\d{4})/,
    /(\w+ \d{1,2}, \d{4})/,
  ];

  for (const pattern of patterns) {
    const match = dateStr.match(pattern);
    if (match) {
      const date = new Date(match[1]);
      if (!isNaN(date.getTime())) {
        return date;
      }
    }
  }

  return null;
}
```

### 2. Anthropic Web Search Tool (Wrapper)

**Dependencies:**
- `@ai-sdk/anthropic`: Official Anthropic provider for Vercel AI SDK
- `ai`: Vercel AI SDK core

**Implementation:**

```typescript
// web-search.ts
import { anthropic } from '@ai-sdk/anthropic';
import { generateText } from 'ai';

export interface WebSearchOptions {
  maxUses?: number;
  allowedDomains?: string[];
  blockedDomains?: string[];
  userLocation?: {
    type: 'approximate';
    country?: string;
    region?: string;
    city?: string;
    timezone?: string;
  };
}

export async function searchWeb(
  query: string,
  options: WebSearchOptions = {}
): Promise<SearchResult[]> {
  const webSearchTool = anthropic.tools.webSearch_20250305({
    maxUses: options.maxUses || 5,
    allowedDomains: options.allowedDomains,
    blockedDomains: options.blockedDomains,
    userLocation: options.userLocation,
  });

  try {
    const result = await generateText({
      model: anthropic('claude-opus-4-20250514'),
      prompt: `Search for: ${query}. Return a concise summary of the top ${options.maxUses || 5} most relevant results with their URLs.`,
      tools: {
        web_search: webSearchTool,
      },
    });

    // Extract sources from result
    return parseWebSearchResults(result);
  } catch (error) {
    if (error instanceof Error && error.message.includes('Web search failed')) {
      console.error('Web search error:', error.message);
      return [];
    }
    throw error;
  }
}

function parseWebSearchResults(result: any): SearchResult[] {
  // The AI SDK may return sources in result.sources
  // or we need to parse from the text response
  const results: SearchResult[] = [];

  // Check if sources are provided
  if (result.sources && Array.isArray(result.sources)) {
    return result.sources.map((source: any) => ({
      title: source.title || '',
      url: source.url || '',
      publishedDate: null, // Web search doesn't always provide dates
      source: source.source || 'Web Search',
    }));
  }

  // Otherwise, parse from text (requires pattern matching)
  // This is a fallback and may need refinement
  const urlPattern = /https?:\/\/[^\s]+/g;
  const urls = result.text.match(urlPattern) || [];

  return urls.map((url: string) => ({
    title: '', // Would need more sophisticated parsing
    url,
    publishedDate: null,
    source: 'Web Search',
  }));
}
```

### 3. Tool Integration with Agent System

Both tools need to integrate with the existing LangChain-based agent system:

```typescript
// google-news.ts
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

export const googleNewsSearchTool = tool(
  async ({ query, maxResults = 5 }) => {
    const results = await parseGoogleNewsRSS(query, maxResults);
    return JSON.stringify(results, null, 2);
  },
  {
    name: 'search_google_news',
    description:
      'Search Google News for recent news articles about a specific topic. ' +
      'Returns articles with titles, URLs, and publication dates. ' +
      'Best for: financial news, company announcements, market events, and dated sources.',
    schema: z.object({
      query: z.string().describe('The search query (e.g., "Apple earnings report")'),
      maxResults: z.number().optional().default(5).describe('Maximum number of results to return'),
    }),
  }
);
```

```typescript
// web-search.ts
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

export const anthropicWebSearchTool = tool(
  async ({ query, maxResults = 5, allowedDomains, blockedDomains }) => {
    const results = await searchWeb(query, {
      maxUses: maxResults,
      allowedDomains,
      blockedDomains,
    });
    return JSON.stringify(results, null, 2);
  },
  {
    name: 'search_web',
    description:
      'Search the web for current information on any topic. ' +
      'Returns relevant web pages with URLs and summaries. ' +
      'Best for: real-time data, general research, fact-checking, and broad topics.',
    schema: z.object({
      query: z.string().describe('The search query'),
      maxResults: z.number().optional().default(5),
      allowedDomains: z.array(z.string()).optional().describe('Limit search to specific domains'),
      blockedDomains: z.array(z.string()).optional().describe('Exclude specific domains'),
    }),
  }
);
```

## Complete Code Examples

### google-news.ts (Complete Implementation)

```typescript
import Parser from 'rss-parser';
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

interface SearchResult {
  title: string;
  url: string;
  publishedDate: Date | null;
  source?: string;
}

interface RSSItem {
  title?: string;
  link?: string;
  pubDate?: string;
  isoDate?: string;
}

/**
 * Resolve Google News redirect URL to actual article URL
 */
async function resolveGoogleNewsUrl(url: string): Promise<string> {
  if (!url || !url.includes('news.google.com')) {
    return url;
  }

  try {
    // Follow redirects to get actual URL
    const response = await fetch(url, {
      method: 'HEAD',
      redirect: 'follow',
      signal: AbortSignal.timeout(5000),
    });
    return response.url;
  } catch (error) {
    // If redirect fails, try base64 decoding
    try {
      const urlObj = new URL(url);
      const articleParam = urlObj.searchParams.get('url');
      if (articleParam) {
        return Buffer.from(articleParam, 'base64').toString('utf-8');
      }
    } catch {}

    return url;
  }
}

/**
 * Clean text by removing HTML, decoding entities, and normalizing Unicode
 */
function cleanText(text: string): string {
  if (!text) return text;

  // Remove HTML tags
  text = text.replace(/<[^>]+>/g, '');

  // Decode HTML entities
  const entities: Record<string, string> = {
    '&amp;': '&',
    '&lt;': '<',
    '&gt;': '>',
    '&quot;': '"',
    '&#39;': "'",
    '&apos;': "'",
  };
  text = text.replace(/&[^;]+;/g, entity => entities[entity] || entity);

  // Replace common Unicode characters
  const unicodeReplacements: Record<string, string> = {
    '\u2018': "'",
    '\u2019': "'",
    '\u201c': '"',
    '\u201d': '"',
    '\u2013': '-',
    '\u2014': '-',
    '\u2026': '...',
    '\u00a0': ' ',
    '\u00ae': '(R)',
    '\u2122': '(TM)',
  };

  for (const [unicode, replacement] of Object.entries(unicodeReplacements)) {
    text = text.replace(new RegExp(unicode, 'g'), replacement);
  }

  // Normalize whitespace
  text = text.replace(/\s+/g, ' ').trim();

  return text;
}

/**
 * Parse date string to Date object
 */
function parseDate(dateStr: string | undefined): Date | null {
  if (!dateStr) return null;

  // Try standard Date parsing first
  try {
    const date = new Date(dateStr);
    if (!isNaN(date.getTime())) {
      return date;
    }
  } catch {}

  // Try cleaning RFC 822 format
  try {
    const cleaned = dateStr.replace(/ GMT| \+0000/g, '');
    const date = new Date(cleaned);
    if (!isNaN(date.getTime())) {
      return date;
    }
  } catch {}

  // Try pattern matching as fallback
  const patterns = [
    /(\d{4}-\d{2}-\d{2})/,
    /(\d{1,2}\/\d{1,2}\/\d{4})/,
    /(\w+ \d{1,2}, \d{4})/,
  ];

  for (const pattern of patterns) {
    const match = dateStr.match(pattern);
    if (match) {
      try {
        const date = new Date(match[1]);
        if (!isNaN(date.getTime())) {
          return date;
        }
      } catch {}
    }
  }

  return null;
}

/**
 * Search Google News RSS feed for articles
 */
export async function searchGoogleNews(
  query: string,
  maxResults: number = 5
): Promise<SearchResult[]> {
  const searchUrl = `https://news.google.com/rss/search?q=${encodeURIComponent(query)}&hl=en-US&gl=US&ceid=US:en`;

  try {
    const parser = new Parser<{}, RSSItem>();
    const feed = await parser.parseURL(searchUrl);

    // Get more items than needed in case some fail to resolve
    const items = feed.items.slice(0, maxResults * 2);

    // Parse RSS items
    const results: SearchResult[] = items.map(item => ({
      title: cleanText(item.title || 'No title'),
      url: item.link || '',
      publishedDate: parseDate(item.pubDate || item.isoDate),
      source: 'Google News',
    }));

    // Resolve Google News redirect URLs in parallel
    const resolvedResults = await Promise.all(
      results.map(async (result) => ({
        ...result,
        url: await resolveGoogleNewsUrl(result.url),
      }))
    );

    // Filter out failed results and limit to maxResults
    return resolvedResults
      .filter(r => r.url && r.url !== '')
      .slice(0, maxResults);

  } catch (error) {
    console.error('Google News search error:', error);
    return [];
  }
}

/**
 * LangChain tool for Google News search
 */
export const googleNewsSearchTool = tool(
  async ({ query, maxResults = 5 }) => {
    const results = await searchGoogleNews(query, maxResults);
    return JSON.stringify(results, null, 2);
  },
  {
    name: 'search_google_news',
    description:
      'Search Google News for recent news articles matching a query. ' +
      'Returns articles with titles, URLs, and publication dates. ' +
      'This tool should be used for: recent news, company announcements, ' +
      'earnings reports, market events, and any query requiring dated sources. ' +
      'Example queries: "Apple earnings report", "Tesla stock news", "Federal Reserve interest rates"',
    schema: z.object({
      query: z.string().describe('The search query to send to Google News'),
      maxResults: z
        .number()
        .optional()
        .default(5)
        .describe('Maximum number of results to return (default: 5)'),
    }),
  }
);
```

### web-search.ts (Complete Implementation)

```typescript
import { anthropic } from '@ai-sdk/anthropic';
import { generateText } from 'ai';
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

interface SearchResult {
  title: string;
  url: string;
  publishedDate: Date | null;
  source?: string;
}

interface WebSearchOptions {
  maxUses?: number;
  allowedDomains?: string[];
  blockedDomains?: string[];
  userLocation?: {
    type: 'approximate';
    country?: string;
    region?: string;
    city?: string;
    timezone?: string;
  };
}

/**
 * Search the web using Anthropic's Web Search tool
 */
export async function searchWeb(
  query: string,
  options: WebSearchOptions = {}
): Promise<SearchResult[]> {
  const webSearchTool = anthropic.tools.webSearch_20250305({
    maxUses: options.maxUses || 5,
    allowedDomains: options.allowedDomains,
    blockedDomains: options.blockedDomains,
    userLocation: options.userLocation,
  });

  try {
    const result = await generateText({
      model: anthropic('claude-opus-4-20250514'),
      prompt: `Search for: "${query}". Provide the top ${options.maxUses || 5} most relevant results with their URLs and titles.`,
      tools: {
        web_search: webSearchTool,
      },
    });

    // Parse results from the response
    return parseWebSearchResults(result);

  } catch (error) {
    if (error instanceof Error && error.message.includes('Web search failed')) {
      console.error('Web search error:', error.message);
      return [];
    }
    throw error;
  }
}

/**
 * Parse web search results from AI SDK response
 */
function parseWebSearchResults(result: any): SearchResult[] {
  // Check if sources are directly available
  if (result.sources && Array.isArray(result.sources)) {
    return result.sources.map((source: any) => ({
      title: source.title || 'No title',
      url: source.url || '',
      publishedDate: null,
      source: 'Web Search',
    }));
  }

  // Fallback: parse URLs from text response
  const results: SearchResult[] = [];
  const lines = result.text.split('\n');

  for (const line of lines) {
    // Look for URL patterns
    const urlMatch = line.match(/https?:\/\/[^\s]+/);
    if (urlMatch) {
      results.push({
        title: line.replace(urlMatch[0], '').trim() || 'No title',
        url: urlMatch[0],
        publishedDate: null,
        source: 'Web Search',
      });
    }
  }

  return results;
}

/**
 * LangChain tool for Anthropic Web Search
 */
export const anthropicWebSearchTool = tool(
  async ({ query, maxResults = 5, allowedDomains, blockedDomains }) => {
    const results = await searchWeb(query, {
      maxUses: maxResults,
      allowedDomains,
      blockedDomains,
    });
    return JSON.stringify(results, null, 2);
  },
  {
    name: 'search_web',
    description:
      'Search the web for current information on any topic. ' +
      'Returns relevant web pages with URLs and summaries. ' +
      'This tool should be used for: real-time information, general research, ' +
      'fact-checking, market data, and broad topics. Can be restricted to ' +
      'specific domains for focused searches. ' +
      'Example queries: "current stock price", "latest AI developments", "company overview"',
    schema: z.object({
      query: z.string().describe('The search query'),
      maxResults: z
        .number()
        .optional()
        .default(5)
        .describe('Maximum number of results (default: 5)'),
      allowedDomains: z
        .array(z.string())
        .optional()
        .describe('Limit search to specific domains (e.g., ["bloomberg.com", "reuters.com"])'),
      blockedDomains: z
        .array(z.string())
        .optional()
        .describe('Exclude specific domains from search'),
    }),
  }
);
```

### index.ts (Exports)

```typescript
export { searchGoogleNews, googleNewsSearchTool } from './google-news';
export { searchWeb, anthropicWebSearchTool } from './web-search';
export type { SearchResult } from './models';
```

### models.ts (Shared Types)

```typescript
export interface SearchResult {
  title: string;
  url: string;
  publishedDate: Date | null;
  source?: string;
}

export interface SearchToolInput {
  query: string;
  maxResults?: number;
}
```

## Package Dependencies

Add to `package.json`:

```json
{
  "dependencies": {
    "@ai-sdk/anthropic": "^1.0.0",
    "@langchain/core": "^0.3.0",
    "ai": "^4.0.0",
    "rss-parser": "^3.13.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0"
  }
}
```

## Migration Path

### Phase 1: Setup (Week 1)
1. Install dependencies: `npm install rss-parser @ai-sdk/anthropic ai`
2. Create `src/tools/search/` directory structure
3. Implement shared types in `models.ts`
4. Set up basic utility functions

### Phase 2: Google News Implementation (Week 1-2)
1. Implement RSS parsing with `rss-parser`
2. Implement URL resolution (HTTP redirect following)
3. Implement text cleaning utilities
4. Implement date parsing utilities
5. Create LangChain tool wrapper
6. Write unit tests

### Phase 3: Web Search Implementation (Week 2)
1. Configure Anthropic provider
2. Implement web search function
3. Implement result parsing
4. Create LangChain tool wrapper
5. Write unit tests

### Phase 4: Integration (Week 3)
1. Update agent configuration to include both tools
2. Test tools with sample queries
3. Update documentation
4. Add error handling and logging

### Phase 5: Testing & Refinement (Week 3-4)
1. Integration testing with agent
2. Performance testing
3. Error handling validation
4. Compare results with Python implementation
5. Documentation updates

## Trade-offs Analysis

### Google News RSS

**Pros:**
- ✅ Reliable publication dates
- ✅ News-specific results
- ✅ No API key required (uses public RSS)
- ✅ Structured data format
- ✅ Good for financial news
- ✅ Direct control over implementation

**Cons:**
- ❌ Limited to news sources
- ❌ Google News redirect URL resolution adds complexity
- ❌ No native googlenewsdecoder in TypeScript
- ❌ RSS parsing overhead
- ❌ May be rate-limited by Google

### Anthropic Web Search

**Pros:**
- ✅ Broader web coverage
- ✅ Real-time information
- ✅ Built-in with Anthropic provider
- ✅ Natural language understanding
- ✅ Can filter by domains
- ✅ Location-aware results

**Cons:**
- ❌ Requires Anthropic API key and organization settings
- ❌ Less predictable date information
- ❌ Uses LLM tokens (cost consideration)
- ❌ May return less structured data
- ❌ Rate limits tied to API usage

### Hybrid Approach

**Pros:**
- ✅ Best of both worlds
- ✅ Redundancy and fallback options
- ✅ Specialized tools for different needs
- ✅ Cross-validation of information
- ✅ Flexibility in agent tool selection

**Cons:**
- ❌ More complex codebase
- ❌ More dependencies to maintain
- ❌ Agent needs to choose appropriate tool
- ❌ Potentially higher API costs

## Recommended Tool Selection Logic

The agent should choose tools based on query characteristics:

**Use Google News when:**
- Query mentions "news", "recent", "latest"
- Query is about specific companies or events
- Date accuracy is critical
- Financial news is needed

**Use Web Search when:**
- Query needs real-time data
- Query is research-oriented
- Query needs broad coverage
- Specific domains are targeted

**Example Decision Logic:**
```typescript
function selectSearchTool(query: string): 'google_news' | 'web_search' {
  const newsKeywords = ['news', 'article', 'report', 'announcement', 'earnings'];
  const realtimeKeywords = ['current', 'now', 'today', 'price', 'data'];

  const lowerQuery = query.toLowerCase();

  if (newsKeywords.some(kw => lowerQuery.includes(kw))) {
    return 'google_news';
  }

  if (realtimeKeywords.some(kw => lowerQuery.includes(kw))) {
    return 'web_search';
  }

  // Default to web search for broader coverage
  return 'web_search';
}
```

## Configuration Requirements

### Environment Variables

```bash
# Required for Anthropic Web Search
ANTHROPIC_API_KEY=your_api_key_here

# Optional: Default search settings
DEFAULT_MAX_RESULTS=5
ENABLE_GOOGLE_NEWS=true
ENABLE_WEB_SEARCH=true
```

### Anthropic Console Setup

For Web Search to work, ensure:
1. Web search is enabled in Anthropic Console organization settings
2. API key has appropriate permissions
3. Rate limits are configured appropriately

## Testing Strategy

### Unit Tests

```typescript
// google-news.test.ts
describe('Google News Search', () => {
  test('should parse RSS feed', async () => {
    const results = await searchGoogleNews('Apple earnings', 5);
    expect(results).toHaveLength(5);
    expect(results[0]).toHaveProperty('title');
    expect(results[0]).toHaveProperty('url');
    expect(results[0]).toHaveProperty('publishedDate');
  });

  test('should resolve Google News URLs', async () => {
    const url = 'https://news.google.com/...';
    const resolved = await resolveGoogleNewsUrl(url);
    expect(resolved).not.toContain('news.google.com');
  });

  test('should clean text properly', () => {
    const dirty = 'Test &amp; &#39;quoted&#39; text';
    const clean = cleanText(dirty);
    expect(clean).toBe("Test & 'quoted' text");
  });
});
```

### Integration Tests

```typescript
describe('Search Tools Integration', () => {
  test('should work with LangChain agent', async () => {
    const agent = createAgent([googleNewsSearchTool, anthropicWebSearchTool]);
    const result = await agent.invoke('Search for recent Apple news');
    expect(result).toBeDefined();
  });
});
```

## Performance Considerations

### Parallel URL Resolution
```typescript
// Batch URL resolution for better performance
const resolved = await Promise.all(
  results.map(r => resolveGoogleNewsUrl(r.url))
);
```

### Caching Strategy
```typescript
// Consider caching RSS feed results
const cache = new Map<string, { results: SearchResult[]; timestamp: number }>();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

async function cachedSearchGoogleNews(query: string, maxResults: number) {
  const cacheKey = `${query}:${maxResults}`;
  const cached = cache.get(cacheKey);

  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.results;
  }

  const results = await searchGoogleNews(query, maxResults);
  cache.set(cacheKey, { results, timestamp: Date.now() });

  return results;
}
```

## Error Handling

### Graceful Degradation

```typescript
export async function searchWithFallback(
  query: string,
  maxResults: number = 5
): Promise<SearchResult[]> {
  try {
    // Try Google News first
    const results = await searchGoogleNews(query, maxResults);
    if (results.length > 0) return results;
  } catch (error) {
    console.error('Google News search failed:', error);
  }

  try {
    // Fallback to Web Search
    return await searchWeb(query, { maxUses: maxResults });
  } catch (error) {
    console.error('Web search failed:', error);
    return [];
  }
}
```

## Monitoring & Logging

```typescript
interface SearchMetrics {
  tool: 'google_news' | 'web_search';
  query: string;
  resultCount: number;
  duration: number;
  success: boolean;
  error?: string;
}

function logSearchMetrics(metrics: SearchMetrics) {
  // Log to monitoring service
  console.log('Search metrics:', metrics);
}
```

## Conclusion

The hybrid approach provides the best solution for Dexter's financial research needs:

1. **Google News RSS** provides specialized, dated news sources critical for financial analysis
2. **Anthropic Web Search** provides broader coverage and real-time information
3. Both tools complement each other and provide fallback options
4. The TypeScript implementation maintains feature parity with Python while leveraging modern async patterns

The recommended implementation prioritizes reliability, performance, and maintainability while providing flexibility for different search scenarios.
