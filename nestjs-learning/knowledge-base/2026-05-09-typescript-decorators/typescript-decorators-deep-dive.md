# TypeScript Decorators Deep Dive (with NestJS Custom Decorators)

## 1) Visual Mental Model

```text
Class Definition Time (not request time)

  TypeScript class + decorators
            |
            v
   Decorator functions execute
            |
            v
   Metadata/descriptor changes are applied
            |
            v
   Runtime behavior is influenced
   (Nest reads metadata for routing/guards/etc.)
```

```text
Nest usage chain

@Roles('admin')  ---> metadata key/value stored
                          |
                          v
                    Reflector.get(...)
                          |
                          v
                    RolesGuard decision
```

## 2) Express vs Spring Boot vs NestJS (Decorator Context)

| Aspect | ExpressJS | Spring Boot | NestJS |
|---|---|---|---|
| Route declaration | Imperative (`router.get`) | Annotation-based (`@GetMapping`) | Decorator-based (`@Get`) |
| Authorization metadata | Usually custom middleware config | Annotation + AOP/security | Decorators + Guard + Reflector |
| Composition style | Manual wiring | IoC container | IoC container |
| Discoverability | Lower in large apps | High | High |

## 3) What a Decorator Is

A decorator is a function attached to a class declaration or class member.
It can observe, annotate, or modify behavior depending on decorator type.

Important: In TypeScript, decorator support requires compiler configuration.

## 4) Required TS Config

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

- `experimentalDecorators`: enables decorators syntax.
- `emitDecoratorMetadata`: emits design-time type metadata used by many Nest patterns.

## 5) Decorator Types and Signatures

### Class Decorator

```ts
type ClassDecorator = <TFunction extends Function>(target: TFunction) => TFunction | void;
```

### Method Decorator

```ts
type MethodDecorator = (
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor,
) => PropertyDescriptor | void;
```

### Accessor Decorator

```ts
type AccessorDecorator = (
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor,
) => PropertyDescriptor | void;
```

### Property Decorator

```ts
type PropertyDecorator = (target: object, propertyKey: string | symbol) => void;
```

### Parameter Decorator

```ts
type ParameterDecorator = (
  target: object,
  propertyKey: string | symbol | undefined,
  parameterIndex: number,
) => void;
```

## 6) Evaluation Order (Very Important)

```text
Within a class:
1) Parameter decorators
2) Method / Accessor / Property decorators
   - instance members first
   - static members next
3) Constructor parameter decorators
4) Class decorator last
```

Why this matters: multi-decorator stacks can produce surprising outcomes if order assumptions are wrong.

## 7) Decorator Factory Pattern

A decorator factory is a function returning a decorator.

```ts
function Tag(label: string): MethodDecorator {
  return (target, key, descriptor) => {
    Reflect.defineMetadata('tag', label, descriptor.value!);
  };
}

class Example {
  @Tag('billing')
  run() {
    return 'ok';
  }
}
```

Use factories when a decorator needs parameters.

## 8) Build Your Own TypeScript Decorators

### Readonly Method Decorator

```ts
export function Readonly(): MethodDecorator {
  return (_target, _key, descriptor: PropertyDescriptor) => {
    descriptor.writable = false;
  };
}
```

### Timing Decorator

```ts
export function Measure(label?: string): MethodDecorator {
  return (_target, key, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = function (...args: unknown[]) {
      const start = performance.now();
      const result = original.apply(this, args);
      const end = performance.now();
      const name = label ?? String(key);
      console.log(`[Measure] ${name}: ${(end - start).toFixed(2)}ms`);
      return result;
    };

    return descriptor;
  };
}
```

## 9) NestJS Custom Decorators

### A) Metadata Decorator with `SetMetadata`

```ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

Guard consumption:

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const userRoles: string[] = request.user?.roles ?? [];
    return requiredRoles.some((r) => userRoles.includes(r));
  }
}
```

### B) Parameter Decorator with `createParamDecorator`

```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (field: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return field ? user?.[field] : user;
  },
);
```

Controller usage:

```ts
@Get('me')
getMe(@CurrentUser() user: { id: string; email: string }) {
  return user;
}

@Get('my-email')
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

## 10) Senior-Level Design Guidance

```text
Good decorator use:
  - Cross-cutting concerns
  - Declarative policy metadata

Bad decorator use:
  - Hiding complex business logic
  - Side effects that make behavior non-obvious
```

- Keep business rules inside services/providers.
- Use decorators to describe intent, not to run domain workflows.
- If a decorator grows too smart, move logic to Guard/Interceptor/Service.

## 11) Common Pitfalls

1. Forgetting `experimentalDecorators`.
2. Assuming property decorators can change initializer value directly.
3. Overusing reflection metadata without clear keys/constants.
4. Coupling decorators to transport context in non-HTTP layers.

## 12) Practice Tasks

1. Implement `@Public()` decorator and skip auth guard when present.
2. Implement `@RateLimit(bucket)` decorator and read it inside an interceptor.
3. Implement `@CurrentTenant()` parameter decorator for multi-tenant apps.

## Documentation Notes

This note is based on:
- TypeScript decorators reference and evaluation order from TypeScript docs
  (`/microsoft/typescript-website` via Context7).
- NestJS custom decorator and metadata/guard usage patterns
  (`/nestjs/nest` via Context7).
