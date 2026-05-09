# NestJS Architecture Deep Dive

## Relationship Model
- Module defines boundaries and dependency visibility.
- Controller handles inbound transport (HTTP/web) and delegates work.
- Provider contains business logic and can depend on other providers.

## Runtime Flow
1. App bootstrap starts with root module.
2. Nest container builds module graph.
3. Providers are instantiated and injected.
4. Controllers receive requests and call providers.

## Why This Matters
- Predictable architecture for large teams
- Easier testing via provider mocking
- Better scalability through bounded module design
