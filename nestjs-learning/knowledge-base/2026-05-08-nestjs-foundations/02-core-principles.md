# Core Principles Behind NestJS

## 1) Modularity
Applications are organized into modules, each owning a focused part of the domain.

## 2) Dependency Injection (DI)
Dependencies are provided by a container instead of being manually instantiated.

## 3) Separation of Concerns
Controllers handle transport concerns; providers handle business logic; modules define boundaries.

## 4) Explicit Architecture
Decorators and metadata make structure visible and enforceable across teams.

## 5) Testability by Design
Because dependencies are injected and concerns are separated, unit/integration tests are easier to write and maintain.
