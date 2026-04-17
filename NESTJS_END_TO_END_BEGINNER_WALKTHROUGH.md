# NestJS End-to-End Beginner Walkthrough

This guide teaches you how to create a real NestJS API from scratch and understand what each part does using beginner-friendly analogies.

---

## What you will build

A simple **Tasks API** with:
- `GET /tasks` (list tasks)
- `GET /tasks/:id` (get one task)
- `POST /tasks` (create task)
- `PATCH /tasks/:id` (update task)
- `DELETE /tasks/:id` (delete task)

---

## Big picture (with analogy)

Think of a NestJS app like a **restaurant**:

- **Module** = a department (kitchen, cashier, delivery)
- **Controller** = waiter taking customer requests
- **Service** = chef doing the real work
- **Provider** = any staff/tool Nest can inject where needed
- **Dependency Injection** = manager assigning the right staff automatically

So when a client calls your API:
1. Request goes to the **Controller** (waiter)
2. Controller asks **Service** (chef) to do the job
3. Service returns result
4. Controller sends response back

---

## Beginner deep dive (topic-by-topic with analogies)

If you feel "I can copy this, but I still don't *get* it", this section is for you.

### 1) Controller (the front desk)

**What it is:** The class that receives HTTP requests and returns responses.  
**Why it exists:** To map URLs and HTTP methods (`GET`, `POST`, etc.) to your code.  
**Analogy:** A hotel front desk. Guests ask for something; front desk routes the request to the right team.

In this guide:
- `@Controller('tasks')` means this desk handles `/tasks/*`
- `@Get()`, `@Post()`, `@Patch()`, `@Delete()` are "which type of request" labels

### 2) Service / Provider (the worker team)

**What it is:** A class with business logic.  
**Why it exists:** Keep controllers thin and focused on request/response flow.  
**Analogy:** Front desk does not cook food; kitchen staff does. Service = kitchen staff.

In this guide:
- `TasksService` stores, finds, updates, and deletes tasks.
- It is marked with `@Injectable()` so Nest can manage and inject it.

### 3) Dependency Injection (automatic wiring)

**What it is:** Nest automatically gives a class the dependencies it needs.  
**Why it exists:** You avoid manually creating objects everywhere.
**Analogy:** You open your workstation and your tools are already there; you don't go buy a new hammer every request.

In this guide:
- `constructor(private readonly tasksService: TasksService) {}`
- Nest sees `TasksService` type and injects the right instance.

### 4) Module (feature container)

**What it is:** A boundary that groups related controller + service + providers.  
**Why it exists:** Structure, scalability, and clean feature ownership.  
**Analogy:** A store inside a mall. The mall (app) has many stores (modules), each with its own staff.

In this guide:
- `TasksModule` is the container for the tasks feature.

### 5) Decorators (instruction stickers)

**What it is:** Metadata annotations like `@Controller`, `@Get`, `@Injectable`.  
**Why it exists:** Tell Nest how to route requests and build dependencies.  
**Analogy:** Stickers on folders: "Finance", "Urgent", "Archive" tell people how to process each folder.

### 6) DTO + ValidationPipe (input quality gate)

**What it is:** DTO classes define allowed input shape; `ValidationPipe` enforces it.  
**Why it exists:** Prevent bad input from reaching business logic.  
**Analogy:** Airport security screening before passengers enter the plane.

In this guide:
- DTO says `title` must be a string with length 3-100.
- If invalid, request is rejected before service logic runs.

### 7) ParseIntPipe (type converter)

**What it is:** A pipe that converts route param strings into numbers.  
**Why it exists:** Route params arrive as strings by default (`"1"`), but your logic expects `1`.  
**Analogy:** Currency exchange booth converting dollars to pesos before shopping.

In this guide:
- `@Param('id', ParseIntPipe) id: number`

### 8) Exceptions (controlled failure)

**What it is:** Framework-friendly errors (`NotFoundException`, etc.)  
**Why it exists:** Return clean, meaningful HTTP error responses.  
**Analogy:** Instead of system crash, the front desk politely says: "Room not found."

In this guide:
- `throw new NotFoundException(...)` gives a proper 404 response.

---

## Request lifecycle (simple mental movie)

When calling `PATCH /tasks/1`, think:

1. Router finds matching controller method.
2. Pipes run (`ParseIntPipe`, `ValidationPipe`) to clean/validate input.
3. Controller method executes.
4. Controller calls service.
5. Service returns result (or throws exception).
6. Nest sends JSON response.

If validation fails at step 2, the request stops there and never reaches your service.  
That is why validation in Nest is powerful and safe by default.

---

## Prerequisites

Install:
- Node.js (LTS recommended)
- npm (comes with Node)

Check versions:

```bash
node -v
npm -v
```

Install Nest CLI:

```bash
npm install -g @nestjs/cli
```

---

## Step 1) Create your NestJS project

```bash
nest new nestjs-tasks-api
cd nestjs-tasks-api
```

Choose `npm` when prompted (or your preferred package manager).

Start dev server:

```bash
npm run start:dev
```

Open `http://localhost:3000`.  
You should see: `Hello World!`

---

## Step 2) Generate a Tasks feature

Create module, controller, and service:

```bash
nest g module tasks
nest g controller tasks --no-spec
nest g service tasks --no-spec
```

Now your app has a new "department" (`tasks`) with its waiter (controller) and chef (service).

---

## Step 3) Create task model and DTOs

Install validation packages:

```bash
npm install class-validator class-transformer
```

Create these files:

```bash
mkdir -p src/tasks/dto
touch src/tasks/task.interface.ts
touch src/tasks/dto/create-task.dto.ts
touch src/tasks/dto/update-task.dto.ts
```

### `src/tasks/task.interface.ts`

```ts
export interface Task {
  id: number;
  title: string;
  completed: boolean;
}
```

### `src/tasks/dto/create-task.dto.ts`

```ts
import { IsString, Length } from 'class-validator';

export class CreateTaskDto {
  @IsString()
  @Length(3, 100)
  title: string;
}
```

### `src/tasks/dto/update-task.dto.ts`

```ts
import { IsBoolean, IsOptional, IsString, Length } from 'class-validator';

export class UpdateTaskDto {
  @IsOptional()
  @IsString()
  @Length(3, 100)
  title?: string;

  @IsOptional()
  @IsBoolean()
  completed?: boolean;
}
```

---

## Step 4) Implement the service (business logic)

### `src/tasks/tasks.service.ts`

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { Task } from './task.interface';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Injectable()
export class TasksService {
  private tasks: Task[] = [];
  private nextId = 1;

  findAll(): Task[] {
    return this.tasks;
  }

  findOne(id: number): Task {
    const task = this.tasks.find((t) => t.id === id);
    if (!task) throw new NotFoundException(`Task ${id} not found`);
    return task;
  }

  create(dto: CreateTaskDto): Task {
    const task: Task = {
      id: this.nextId++,
      title: dto.title,
      completed: false,
    };
    this.tasks.push(task);
    return task;
  }

  update(id: number, dto: UpdateTaskDto): Task {
    const task = this.findOne(id);
    if (dto.title !== undefined) task.title = dto.title;
    if (dto.completed !== undefined) task.completed = dto.completed;
    return task;
  }

  remove(id: number): void {
    const before = this.tasks.length;
    this.tasks = this.tasks.filter((t) => t.id !== id);
    if (this.tasks.length === before) {
      throw new NotFoundException(`Task ${id} not found`);
    }
  }
}
```

---

## Step 5) Implement the controller (routes)

### `src/tasks/tasks.controller.ts`

```ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  ParseIntPipe,
  Patch,
  Post,
} from '@nestjs/common';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Controller('tasks')
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Get()
  findAll() {
    return this.tasksService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.tasksService.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateTaskDto) {
    return this.tasksService.create(dto);
  }

  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() dto: UpdateTaskDto,
  ) {
    return this.tasksService.update(id, dto);
  }

  @Delete(':id')
  remove(@Param('id', ParseIntPipe) id: number) {
    this.tasksService.remove(id);
    return { message: `Task ${id} deleted` };
  }
}
```

---

## Step 6) Enable global validation

### `src/main.ts`

Make sure it includes:

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      transform: true,
      forbidNonWhitelisted: true,
    }),
  );
  await app.listen(3000);
}
bootstrap();
```

Why this matters:
- `whitelist: true` removes unexpected fields
- `forbidNonWhitelisted: true` throws errors for unknown fields
- `transform: true` helps convert input types safely

---

## Step 7) Run and test the API

Start server:

```bash
npm run start:dev
```

Test with `curl`:

Create task:

```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn NestJS basics"}'
```

Get all tasks:

```bash
curl http://localhost:3000/tasks
```

Update task:

```bash
curl -X PATCH http://localhost:3000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed":true}'
```

Delete task:

```bash
curl -X DELETE http://localhost:3000/tasks/1
```

---

## Step 8) What to learn next

Follow the comprehensive sequential guide:

**→ [NESTJS_NEXT_STEPS_COMPLETE_GUIDE.md](./NESTJS_NEXT_STEPS_COMPLETE_GUIDE.md)**

This single guide walks you through:
1. **Part 1: Database** - Add PostgreSQL + TypeORM persistence
2. **Part 2: Configuration** - Use .env files and ConfigService
3. **Part 3: Authentication** - Add JWT login and protected routes
4. **Testing** - Complete end-to-end testing with curl

By the end, you'll have a **production-ready API** with database, config, and authentication!

---

## Quick beginner mental model

- **Controller** = "Where requests enter"
- **Service** = "Where logic lives"
- **Module** = "How features are grouped"
- **DTO** = "Input contract and validation rules"
- **Pipe** = "Input cleaner/validator before logic"

If you remember this flow, you can read and build most NestJS APIs with confidence.
