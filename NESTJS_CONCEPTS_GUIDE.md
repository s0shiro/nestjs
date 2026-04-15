# NestJS Core Concepts - Learning Guide

A condensed guide covering the 10 essential NestJS concepts for building solid foundations.

---

## Table of Contents

1. [Application](#1-application)
2. [Modules](#2-modules)
3. [Decorators](#3-decorators)
4. [Controllers](#4-controllers)
5. [Providers](#5-providers)
6. [Middlewares](#6-middlewares)
7. [Guards](#7-guards)
8. [Interceptors](#8-interceptors)
9. [Pipes](#9-pipes)
10. [Filters](#10-filters)

---

## Introduction

Understanding these 10 core concepts will allow you to navigate any NestJS code you encounter. You don't need to memorize everything, but understanding the **key principles** is essential for becoming a competent NestJS developer.

---

## 1. Application

NestJS is not limited to building APIs. There are **three types of applications** you can build:

### HTTP Server (REST/Web APIs)
- Created using `NestFactory.create()`
- Handles HTTP requests and returns responses
- Used for building traditional REST APIs and web servers

### Microservices
- Created using `NestFactory.createMicroservice()`
- Similar to HTTP servers but use different transport protocols (TCP, AMQP, etc.)
- Communicate via internal networks
- Ideal for distributed system architectures

### Standalone Applications
- Created using `NestFactory.createApplicationContext()`
- Don't have a network listener
- Perfect for scheduled tasks (cron jobs), CLI tools, or background workers

**Common Point:** All three types require a **root module** to bootstrap the application.

---

## 2. Modules

Modules are the **building blocks** of a NestJS application.

### Module Structure
- A NestJS application is a **graph made of modules**
- Every application has a **root module** at the top
- Modules can import other modules, and those can import more modules (tree-like structure)

### Module Example
```
Root Module
├── Product Module
├── Order Module
│   ├── Cart Module
│   └── Payment Module
└── Configuration Module (shared)
```

### Implementation
- A module is a **class annotated with `@Module()` decorator**
- You define relationships between modules using the `imports` property
- Modules encapsulate controllers, providers, and other logic

---

## 3. Decorators

Decorators are **special functions** that modify the behavior of classes, methods, properties, and can even add metadata.

### What Are Decorators?

Think of decorators as **accessories or suits** that give objects extra power and capabilities.

### Decorator Capabilities
- Attach to: classes, methods, properties
- Modify behavior
- Add metadata to enhance functionality
- Enable powerful features when combined

### Example: Module Decorator
```
Naked Class + @Module() decorator = 
Module class that can be imported by other modules
```

### Common NestJS Decorators
- `@Module()` - Marks a class as a module
- `@Controller()` - Marks a class as a controller
- `@Injectable()` - Marks a class as a provider
- `@Get()`, `@Post()`, etc. - HTTP verb decorators
- `@Guard()`, `@Interceptor()`, etc. - Middleware decorators

---

## 4. Controllers

Controllers are classes that **receive incoming requests and return responses**.

### Responsibilities
- Accept HTTP requests
- Return HTTP responses
- Delegate business logic to other services (providers)

### Structure
- Annotated with `@Controller()` decorator
- Define a root path (e.g., `/users`)
- Contain handler methods for different routes

### Handler Methods
- Each method handles a specific HTTP endpoint
- Decorated with HTTP verb decorators: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, etc.
- Can accept a sub-route as parameter

### Key Principle
> The main role of the controller is to receive requests and return responses. Everything else is delegated to other classes.

---

## 5. Providers

Providers are classes that can be **injected into other classes as dependencies**.

### What Is a Provider?
- A class annotated with `@Injectable()` decorator
- Contains business logic, data access, utility functions, etc.
- Can be injected into controllers or other providers

### Dependency Injection
1. Create a service class and annotate it with `@Injectable()`
2. Declare it as a provider in the module
3. Inject it in the controller via the constructor

### Analogy
**Providers = Freelance Workers**
**NestJS = Agency**

NestJS is an agency that places freelance workers (providers) wherever they are needed.

### Patterns Used
- **Dependency Injection**: Automatically provides dependencies
- **Singleton Pattern**: Each provider exists as a single instance throughout the application

---

## 6. Middlewares

Middlewares allow requests to go through **multiple processing stages** before reaching the handler.

### Request Journey Analogy
Like taking a plane, requests go through stages:
1. Check-in (validation)
2. Security (authentication)
3. Border control (authorization)
4. Boarding pass check (verification)
5. Final boarding (handler)

### Common Use Cases
- Logging every incoming request
- Request data transformation
- Authentication/Authorization preprocessing
- Adding headers or metadata

### Important Note
> Don't add middlewares everywhere. NestJS provides built-in features (Guards, Interceptors) for common request flow management.

---

## 7. Guards

Guards are **security checkpoints** in the request lifecycle that determine whether a request should proceed.

### Primary Purpose
Determine whether a request will be handled by the route handler based on **specific conditions**:
- Roles
- Permissions
- Custom logic
- Any other condition

### Capabilities
- Access the execution context
- Inspect request details
- Check authentication status
- Verify authorization

### Guard Behavior
- **Returns `true`**: Request passes through to handler
- **Returns `false`**: Request is denied, error response sent

### Common Use Case
An authentication guard checks if the user is authenticated and authorized to access the requested resource.

---

## 8. Interceptors

Interceptors give you **full control over the request-response cycle** by running before and after the route handler.

### Capabilities
- Execute code before the handler
- Execute code after the handler
- Intercept request/response data

### Common Use Cases
- Logging request/response details
- Caching responses
- Transforming response format
- Measuring request duration
- Global error handling

### Interface
- Implement `NestInterceptor` interface
- Get full control over the request-response pipeline

---

## 9. Pipes

Pipes validate and **transform data** before it reaches the handler.

### Two Main Functions

#### 1. Validation
- Validate incoming data against expected format/type
- Check constraints and rules
- Throw exceptions if validation fails (prevents handler execution)

#### 2. Transformation
- Convert data into suitable format for processing
- Examples: string → integer, parse dates from strings, etc.

### Built-in Pipes
- `ValidationPipe` - Validates data structure
- `ParseIntPipe` - Converts to integer
- `ParseUUIDPipe` - Validates UUID format
- Others for common transformations

### Custom Pipes
Create custom pipes for specific requirements by implementing `PipeTransform` interface.

---

## 10. Filters

Exception Filters provide a **centralized way to handle errors** and maintain application stability.

### What They Do
- Catch exceptions thrown during request handling
- Determine how to handle the exception
- Send consistent error responses to clients

### Where They Catch Errors
Exception filters can catch errors from:
- Guards
- Interceptors
- Pipes
- Route handlers
- Any other part of request handling

### Default Behavior
- NestJS returns 500 (internal server error) by default
- You can throw standard HTTP exceptions with more detail

### Benefits of Global Exception Filters
- Centralized error handling
- Consistent error messages for clients
- Prevent sensitive information exposure
- Makes frontend developers happy with predictable error responses
- Maintains application stability and security

### Implementation
- Classes implement `ExceptionFilter` interface
- Decorated with `@Catch()` decorator
- Can be applied globally or to specific routes

---

## Request Lifecycle Summary

Here's the order in which NestJS processes a request:

```
1. Middleware (pre-processing)
   ↓
2. Guards (authorization checks)
   ↓
3. Interceptors (pre-handler)
   ↓
4. Pipes (validation & transformation)
   ↓
5. Route Handler (controller method)
   ↓
6. Interceptors (post-handler)
   ↓
7. Filters (catch any exceptions)
   ↓
8. Response sent to client
```

---

## Key Takeaways

1. **Applications**: Not just APIs - can be HTTP servers, microservices, or standalone apps
2. **Modules**: Building blocks organized hierarchically with a root module
3. **Decorators**: Special functions that enhance classes and methods
4. **Controllers**: Handle requests and delegate logic to providers
5. **Providers**: Contain business logic and are injected via dependency injection
6. **Middlewares**: Pre-processing stage for all requests
7. **Guards**: Authorization checkpoints to allow/deny requests
8. **Interceptors**: Full control over request-response cycle
9. **Pipes**: Validate and transform incoming data
10. **Filters**: Centralized error handling and responses

---

## Next Steps

To solidify these concepts:
- Apply them in practice by building a real API
- Follow the progression: Application → Modules → Decorators → Controllers → Providers
- Implement the request lifecycle features incrementally
- Reference this guide when building NestJS applications

**Remember**: You don't need to memorize everything. Focus on understanding the **principles and how they work together** to create a robust application framework.
