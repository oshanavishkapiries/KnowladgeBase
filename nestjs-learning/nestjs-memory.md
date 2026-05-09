# NestJS Learning Memory — Persistent Context File

> Auto-maintained by AI Agent | Language: English
> Learner Goal: Master NestJS at Senior Engineer Level
> Last Updated: 2024-05-22

---

## Learner Profile
- **Background**: 4+ years JavaScript & TypeScript experience
- **Current JS/TS Level**: Intermediate-Advanced  
- **Previous Backend Experience**: ExpressJS (4+ years, production level)
- **Comparison Framework**: Learns best via ExpressJS vs Spring Boot vs NestJS
- **Learning Style**: 
  - VISUAL learner — needs ASCII diagrams for every concept
  - Diagram first, explanation after
  - Comparison-based learning (Express → Spring Boot → NestJS)
- **TypeScript Decorators**: Beginner — needs deep foundation here
- **Learning Goal**: Senior-level NestJS mastery
- **Session 1 Date**: 2025-07-13
---

## Learning Strategy Adjustments (Based on Profile)
1. **Skip Basic JS/TS Concepts**: Assume full proficiency in async/await, promises, generics, decorators, types.
2. **Express Bridge Strategy**: 
   - Always map NestJS concepts to Express equivalents
   - Highlight what NestJS abstracts vs what remains the same
   - Show how to use Express middleware/patterns in NestJS when needed
3. **Focus Areas**:
   - Architecture & Dependency Injection (main shift from Express)
   - Advanced Patterns (CQRS, Event-Driven, Microservices)
   - Testing Strategies
   - Performance & Scalability
   - Production Engineering

---

## Learning Roadmap

### Phase 1 — Core Fundamentals & Express Migration Mindset
- 📋 NestJS Architecture vs Express Mental Model
- 📋 Modules System (Static & Dynamic)
- 📋 Controllers & Routing (Express comparison)
- 📋 Providers & Dependency Injection (NEW CONCEPT)
- 📋 Services
- 📋 Middleware (Express compatibility)

### Phase 2 — Intermediate Concepts
- 📋 Pipes & Validation
- 📋 Guards & Authentication
- 📋 Interceptors
- 📋 Exception Filters
- 📋 Custom Decorators

### Phase 3 — Advanced Patterns
- 📋 Dynamic Modules
- 📋 Module Federation
- 📋 CQRS Pattern
- 📋 Event-Driven Architecture
- 📋 Microservices
- 📋 gRPC / Message Queues

### Phase 4 — Production Engineering
- 📋 Testing Strategy (Unit, Integration, E2E)
- 📋 Performance Optimization
- 📋 Configuration Management
- 📋 Database Patterns (TypeORM / Prisma)
- 📋 Caching Strategies
- 📋 Deployment & Docker

---

## Knowledge Base

### Concepts Mastered
*(Will be populated as learning progresses)*

### Express → NestJS Mappings
*(Key comparisons discovered during learning)*

---

## Session History

### Session 0 — Profile Setup
- **Date**: 2024-05-22
- **Summary**: 
  - Learner profile established
  - 4+ years JS/TS experience confirmed
  - Express.js background confirmed
  - Learning strategy adjusted for senior-level deep dive
  - Memory workflow initialized

---

## Open Questions
*(Questions that need deeper exploration in future sessions)*

---

## Code Examples Index
*(References to saved code examples)*

## Knowledge Artifacts Index
- 2026-05-08: `nestjs-learning/knowledge-base/2026-05-08-nestjs-foundations/README.md`

---

## Key Insights & Patterns Discovered
*(Important "aha moments" and senior-level insights)*

---

## Topics Pending
- 🔄 TypeScript Decorators (deep dive — foundation for everything)

---

## Session History

### Session 1 — NestJS Foundations: What, Why, Problems, Capabilities
- **Date**: 2026-05-08
- **Summary**:
  - Covered what NestJS is and why it was created
  - Explained core principles: modular architecture, dependency injection, separation of concerns
  - Mapped Node ecosystem problem (no standard architecture) to NestJS solution
  - Covered key capabilities: REST, GraphQL, WebSockets, microservices, guards, pipes, interceptors, filters
  - Compared ExpressJS vs Spring Boot vs NestJS mental model
- **Progress Update**:
  - ✅ NestJS introduction (what/why/problem/capabilities)
  - 🔄 Next: NestJS Architecture deep dive (Module, Controller, Provider relationship)

## Concepts Mastered
- ✅ NestJS purpose and design philosophy
- ✅ Why NestJS solves backend architecture standardization problems in Node.js

## Express → NestJS Mappings
- Express app structure (routes/services/middleware by convention) → NestJS explicit architecture via Module/Controller/Provider
- Manual dependency wiring in Express → IoC + DI container in NestJS

## Open Questions
- How exactly Module, Controller, and Provider collaborate at runtime through the DI container?

---

### Session 2 — TypeScript Decorators Deep Dive
- **Date**: 2026-05-09
- **Summary**:
  - Covered what decorators are and why they exist in TypeScript
  - Covered decorator types: class, method, accessor, property, parameter
  - Covered execution/evaluation order and descriptor-based behavior changes
  - Covered compiler flags: experimentalDecorators and emitDecoratorMetadata
  - Covered NestJS custom decorator patterns with SetMetadata and createParamDecorator
- **Artifacts**:
  - `nestjs-learning/knowledge-base/2026-05-09-typescript-decorators/typescript-decorators-deep-dive.md`
- **Progress Update**:
  - ✅ TypeScript decorators foundation and custom decorator creation
  - 🔄 Next: implement decorators in a mini feature module (guard + interceptor + param decorator)

## Knowledge Artifacts Index
- 2026-05-09: `nestjs-learning/knowledge-base/2026-05-09-typescript-decorators/typescript-decorators-deep-dive.md`
