# Express vs Spring Boot vs NestJS

| Aspect | ExpressJS | Spring Boot | NestJS |
|---|---|---|---|
| Framework style | Minimal and unopinionated | Opinionated enterprise framework | Opinionated architecture-first Node framework |
| DI model | Usually manual | Built-in IoC container | Built-in DI container |
| Structure | Convention chosen by team | Standard layered approach | Modules/Controllers/Providers |
| Scaling teams | Depends on discipline | High consistency by default | High consistency by default |
| Testing experience | Variable | Strong | Strong |

## Mental Mapping
- Express route handlers map to Nest controllers.
- Express service files map to Nest providers.
- Manual wiring in Express maps to Nest DI container.
