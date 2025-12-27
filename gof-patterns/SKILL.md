---
name: gof-design-patterns
description: Detects opportunities to apply GoF Design Patterns in Jakarta EE/MicroProfile. Use when reviewing code for structural improvements or implementing creational, structural, or behavioral patterns.
---

# GoF Design Patterns Refactoring Detection

Identify opportunities to apply Gang of Four (GoF) Design Patterns in Jakarta EE and MicroProfile applications.

---

## Creational Patterns

### Factory Method

**Detect**: `if/switch` with `new` based on type/condition  
**Refactor**: CDI `Instance<T>` + `@Named` qualifiers  
**Cookbook**: [factory-method.md](./cookbook/factory-method.md)

### Abstract Factory

**Detect**: Creating families of related objects with scattered logic  
**Refactor**: CDI qualifiers for factory interface implementations  
**Cookbook**: [abstract-factory.md](./cookbook/abstract-factory.md)

### Builder

**Detect**: Constructors with 4+ parameters, telescoping constructors, many setters in sequence  
**Refactor**: Fluent builder with `build()` method, immutable objects  
**Cookbook**: [builder.md](./cookbook/builder.md)

### Singleton

**Detect**: Manual `getInstance()` with double-checked locking  
**Refactor**: `@ApplicationScoped` CDI bean  
**Cookbook**: [singleton.md](./cookbook/singleton.md)

### Prototype

**Detect**: Expensive object creation, need to copy/clone objects  
**Refactor**: `Cloneable` interface + prototype registry  
**Cookbook**: [prototype.md](./cookbook/prototype.md)

---

## Structural Patterns

### Adapter

**Detect**: Interface mismatch with external/legacy APIs  
**Refactor**: CDI bean wrapping legacy system with target interface  
**Cookbook**: [adapter.md](./cookbook/adapter.md)

### Bridge

**Detect**: Abstraction and implementation should vary independently  
**Refactor**: Separate hierarchy for implementors, inject via CDI  
**Cookbook**: [bridge.md](./cookbook/bridge.md)

### Composite

**Detect**: Tree/hierarchy structures (menus, org charts, file systems)  
**Refactor**: Recursive interface, JPA `@OneToMany` self-reference  
**Cookbook**: [composite.md](./cookbook/composite.md)

### Decorator

**Detect**: Inheritance explosion, cross-cutting concerns mixed in business logic  
**Refactor**: CDI `@Decorator` + `@Delegate`, `@Interceptor` for AOP  
**Cookbook**: [decorator.md](./cookbook/decorator.md)

### Facade

**Detect**: Resource/controller calling multiple services directly  
**Refactor**: `@ApplicationScoped` facade service orchestrating subsystems  
**Cookbook**: [facade.md](./cookbook/facade.md)

### Flyweight

**Detect**: Many similar objects causing high memory usage  
**Refactor**: Shared object cache, separate intrinsic/extrinsic state  
**Cookbook**: [flyweight.md](./cookbook/flyweight.md)

### Proxy

**Detect**: Manual lazy loading, scattered access control checks  
**Refactor**: CDI proxies (automatic), `@RolesAllowed`, MicroProfile REST Client  
**Cookbook**: [proxy.md](./cookbook/proxy.md)

---

## Behavioral Patterns

### Chain of Responsibility

**Detect**: Sequential handler processing with hardcoded chain  
**Refactor**: CDI `Instance<T>` + ordering, JAX-RS filters  
**Cookbook**: [chain-of-responsibility.md](./cookbook/chain-of-responsibility.md)

### Command

**Detect**: Operations needing undo/redo, queuing, or logging  
**Refactor**: Command objects with `execute()`/`undo()` + executor  
**Cookbook**: [command.md](./cookbook/command.md)

### Interpreter

**Detect**: DSL/grammar parsing, configurable business rules  
**Refactor**: Expression tree with `interpret()` method  
**Cookbook**: [interpreter.md](./cookbook/interpreter.md)

### Iterator

**Detect**: Custom traversal needed, lazy loading from database  
**Refactor**: Custom `Iterator<T>` for paginated queries  
**Cookbook**: [iterator.md](./cookbook/iterator.md)

### Mediator

**Detect**: Complex inter-object communication, many direct dependencies  
**Refactor**: CDI Events (`Event<T>` + `@Observes`) or mediator service  
**Cookbook**: [mediator.md](./cookbook/mediator.md)

### Memento

**Detect**: Need undo/redo, state snapshots, version history  
**Refactor**: Memento objects + caretaker for history  
**Cookbook**: [memento.md](./cookbook/memento.md)

### Observer

**Detect**: Direct method calls to notify multiple dependents  
**Refactor**: CDI `Event<T>` + `@Observes` / `@ObservesAsync`  
**Cookbook**: [observer.md](./cookbook/observer.md)

### State

**Detect**: Complex conditionals based on object state (`if status == X`)  
**Refactor**: State interface with per-state implementations  
**Cookbook**: [state.md](./cookbook/state.md)

### Strategy

**Detect**: `switch`/`if-else` selecting between algorithms  
**Refactor**: CDI `Instance<T>` + strategy selection by qualifier/type  
**Cookbook**: [strategy.md](./cookbook/strategy.md)

### Template Method

**Detect**: Duplicate algorithm structure with varying steps  
**Refactor**: Abstract class with `final` template + abstract hooks  
**Cookbook**: [template-method.md](./cookbook/template-method.md)

### Visitor

**Detect**: Operations across unrelated element types  
**Refactor**: Visitor interface + `accept(visitor)` in elements  
**Cookbook**: [visitor.md](./cookbook/visitor.md)

---

## Quick Reference Table

| Pattern                 | Detection Signal                       | Jakarta EE/MP Integration        |
| ----------------------- | -------------------------------------- | -------------------------------- |
| Factory Method          | `if/switch` with `new` based on type   | CDI `Instance<T>` + qualifiers   |
| Abstract Factory        | Creating families of related objects   | CDI qualifiers for factory types |
| Builder                 | Constructors with 4+ parameters        | Immutable objects + fluent API   |
| Singleton               | Manual `getInstance()` pattern         | `@ApplicationScoped`             |
| Prototype               | Expensive object creation, need copies | `Cloneable` + prototype registry |
| Adapter                 | Interface mismatch with external APIs  | CDI bean wrapping legacy         |
| Bridge                  | Abstraction/implementation should vary | CDI for implementor injection    |
| Composite               | Tree/hierarchy structures              | Recursive interface + JPA        |
| Decorator               | Inheritance for extending behavior     | `@Decorator`, `@Interceptor`     |
| Facade                  | Multiple service calls in controller   | `@ApplicationScoped` facade      |
| Flyweight               | Many similar objects, high memory      | Shared caches, object pools      |
| Proxy                   | Manual lazy loading / access control   | CDI proxies, `@RolesAllowed`     |
| Chain of Responsibility | Sequential handler processing          | CDI `Instance<T>` + ordering     |
| Command                 | Operations needing undo/queue/log      | Command objects + executor       |
| Interpreter             | DSL/grammar parsing, rule engines      | Expression trees + parser        |
| Iterator                | Custom traversal, lazy DB loading      | `Iterator<T>`, paginated queries |
| Mediator                | Complex inter-object communication     | CDI Events or mediator service   |
| Memento                 | Undo/redo, state snapshots             | Memento objects + caretaker      |
| Observer                | Direct calls to multiple dependents    | CDI `Event<T>` + `@Observes`     |
| State                   | Complex state-based conditionals       | State interface + transitions    |
| Strategy                | `switch`/`if-else` selecting algorithm | CDI `Instance<T>` + iteration    |
| Template Method         | Duplicate algorithm with varying steps | Abstract class with hooks        |
| Visitor                 | Operations across unrelated elements   | Visitor interface + accept       |

---

## MicroProfile-Specific Patterns

| MP Feature      | Pattern Applied    | Example                                              |
| --------------- | ------------------ | ---------------------------------------------------- |
| Health Check    | Factory + Strategy | `@Liveness`, `@Readiness` implementing `HealthCheck` |
| Fault Tolerance | Decorator/Proxy    | `@Retry`, `@Timeout`, `@Fallback` annotations        |
| Config          | Strategy           | `@ConfigProperty` to select behavior at runtime      |
| REST Client     | Proxy              | Interface-based remote service invocation            |
| Metrics         | Decorator          | `@Counted`, `@Timed` interceptors                    |

---

## When NOT to Apply Patterns

1. **Simple cases**: Don't add Factory for 2-3 types that rarely change
2. **YAGNI**: Don't add patterns for "future flexibility"
3. **Performance-critical paths**: Pattern overhead may matter
4. **Clear and readable code**: Patterns shouldn't obscure intent
5. **Team familiarity**: Consider team's pattern knowledge

---

## Cookbook Index

All 23 GoF patterns with Jakarta EE implementations:

**Creational (5)**: [Factory Method](./cookbook/factory-method.md) · [Abstract Factory](./cookbook/abstract-factory.md) · [Builder](./cookbook/builder.md) · [Singleton](./cookbook/singleton.md) · [Prototype](./cookbook/prototype.md)

**Structural (7)**: [Adapter](./cookbook/adapter.md) · [Bridge](./cookbook/bridge.md) · [Composite](./cookbook/composite.md) · [Decorator](./cookbook/decorator.md) · [Facade](./cookbook/facade.md) · [Flyweight](./cookbook/flyweight.md) · [Proxy](./cookbook/proxy.md)

**Behavioral (11)**: [Chain of Responsibility](./cookbook/chain-of-responsibility.md) · [Command](./cookbook/command.md) · [Interpreter](./cookbook/interpreter.md) · [Iterator](./cookbook/iterator.md) · [Mediator](./cookbook/mediator.md) · [Memento](./cookbook/memento.md) · [Observer](./cookbook/observer.md) · [State](./cookbook/state.md) · [Strategy](./cookbook/strategy.md) · [Template Method](./cookbook/template-method.md) · [Visitor](./cookbook/visitor.md)
