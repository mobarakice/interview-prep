# JavaScript & TypeScript Senior Developer Study Guide

This guide covers JavaScript execution mechanics, prototypal inheritance, async patterns, TypeScript type systems (Generics, Mapped, Conditional Types), and TS compiler configuration.

---

## 1. JavaScript Core Execution Mechanics

### Prototypes & Prototypal Inheritance
JavaScript uses prototype-based inheritance. Every object has a private property (`__proto__`) linking to another object called its **Prototype**.
* **Prototype Chain:** When accessing a property on an object, JavaScript checks the object itself. If not found, it traverses the `__proto__` link up the chain until it either finds the property or hits `null`.
* **Performance Concern:** Accessing properties high up the prototype chain can impact performance in hot code paths.

### Closures & Lexical Scope
* **Lexical Scope:** JavaScript resolves variable access based on the physical position of the variable declarations within the source code.
* **Closure:** A function that retains access to its lexical scope (outer variables) even when the function is executed outside its original scope.
  ```javascript
  function createCounter() {
      let count = 0; // Enclosed variable
      return function() {
          return ++count; // Accesses parent lexical scope
      };
  }
  const counter = createCounter();
  console.log(counter()); // 1
  ```

### Event Propagation
* **Capturing Phase:** Event moves down from the window element to the target element.
* **Target Phase:** Event triggers on the target element.
* **Bubbling Phase:** Event bubbles up from the target element back to the window.
* *Mitigation:* Use `event.stopPropagation()` to prevent bubbling, or `event.preventDefault()` to block default browser behaviors.

---

## 2. Asynchronous JavaScript & Promise Combinators

### Promise Combinators Comparison

| Method | Behavior | Use Case |
| :--- | :--- | :--- |
| **`Promise.all`** | Resolves when *all* promises resolve. Rejects immediately if *any* promise rejects. | Parallel independent fetches (all-or-nothing). |
| **`Promise.allSettled`** | Resolves after *all* promises settle (either resolved or rejected). Returns status array. | Bulk operations where failures are handled individually. |
| **`Promise.race`** | Resolves/Rejects as soon as the *first* promise settles. | Timeout patterns or racing fast mirror servers. |
| **`Promise.any`** | Resolves as soon as the *first* promise resolves. Rejects only if *all* promises fail. | Querying multiple fallback CDNs. |

---

## 3. TypeScript Type System: Core & Advanced

### Interfaces vs. Type Aliases
* **Interfaces:** Open for extension. Supports **Declaration Merging** (declaring the same interface twice merges properties). Used to define object shapes and contracts.
* **Type Aliases:** Closed. Cannot be declared twice. Supports unions (`type Status = 'active' | 'inactive'`), intersections, and utility types.

### Structural vs. Nominal Typing
* **Structural Typing (TypeScript):** Type compatibility is determined solely by the shape of the data. If class A has properties `x` and `y`, it is compatible with class B having `x` and `y`.
* **Nominal Typing (Java/C++):** Type compatibility is determined by the explicit class names and inheritance hierarchy.

### Advanced Types Blueprint
```typescript
// 1. Conditional Types
type IsString<T> = T extends string ? true : false;
type StringCheck = IsString<number>; // false

// 2. Mapped Types (Transforming properties)
type ReadOnlyFields<T> = {
  readonly [P in keyof T]: T[P];
};

// 3. Template Literal Types (Dynamic string pattern checking)
type TenantSubdomain = `${string}.im.com`;
const validDomain: TenantSubdomain = "unilever.im.com"; // Valid
```

### Type Guards & Narrowing
Type guards narrow down union types to a specific concrete type within a block:
```typescript
interface TenantUser {
  id: string;
  role: 'tenant';
}
interface AdminUser {
  id: string;
  permissions: string[];
  role: 'admin';
}

// User-Defined Type Guard
function isAdmin(user: TenantUser | AdminUser): user is AdminUser {
  return user.role === 'admin';
}

public async void processUser(user: TenantUser | AdminUser) {
  if (isAdmin(user)) {
    console.log(user.permissions); // Narrowed safely to AdminUser
  }
}
```

---

## 4. TypeScript Compiler Configuration (`tsconfig.json`)

To configure TypeScript projects for production stability:
* `strict: true`: Enables strict type checking flags, including `noImplicitAny`, `strictNullChecks`, and `strictBindCallApply`.
* `target`: Sets the JavaScript language version for emitted JavaScript (e.g. `es2022`).
* `moduleResolution`: Determines how modules are resolved (e.g. `node16` or `bundler`).
* `noUnusedLocals` / `noUnusedParameters`: Raises compiler errors if variables or parameters are declared but unused.

---

### Questions & Answers: JavaScript & TypeScript

#### Q1: What is a closure? Write code to demonstrate how closures can cause memory leaks in JavaScript.
**Answer:**
> "A closure is a function that retains access to variables in its outer lexical scope even after that outer function has returned. 
> Closures cause memory leaks if they retain references to large objects in their parent scope that are no longer needed."
```javascript
function leakMemory() {
  const largeArray = new Array(1000000).fill('data'); // Large array
  
  return function() {
    // This inner function does not use largeArray,
    // but because it shares the parent scope context,
    // largeArray is retained in memory.
    console.log("Closure executed");
  };
}
const closure = leakMemory(); // largeArray is leaked in memory
```

#### Q2: What is the difference between `any`, `unknown`, and `never` in TypeScript?
**Answer:**
> "1. **`any`:** Bypasses type checking. You can perform any operation or read any property on it. Disables TypeScript type safety.
> 2. **`unknown`:** The type-safe counterpart to `any`. Represents any value. You cannot read properties or invoke methods on an `unknown` variable without first narrowing the type (e.g., using type guards or checking `typeof`).
> 3. **`never`:** Represents values that should never occur. Used for functions that throw errors, run infinite loops, or as the fallback type in conditional types to filter out invalid values."

---
JS & TS Study Guide
