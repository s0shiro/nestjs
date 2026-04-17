# NestJS Database Integration - Beginner's Guide

After mastering the in-memory Tasks API, it's time to **persist data** using a real database. This guide teaches you TypeORM + PostgreSQL with beginner-friendly analogies.

---

## What You'll Learn

- Why databases matter
- Setting up PostgreSQL
- Creating entities (database tables as code)
- Using repositories to query data
- Making relationships between tables

---

## The Database Concept (With Analogy)

**Your in-memory API is like a whiteboard:**
- Write data on it → `this.tasks.push(task)`
- Read data from it → `this.tasks.find(...)`
- Problem: When you close the app, everything erases!

**A database is like a filing cabinet:**
- Write to filing cabinet → Data stays even after restart
- Read from filing cabinet → Always there
- Backup your important files → Real data persistence

---

## Step 1: Install Dependencies

```bash
npm install @nestjs/typeorm typeorm pg
```

**What each does:**
- `@nestjs/typeorm` - NestJS integration for TypeORM
- `typeorm` - The actual ORM (Object-Relational Mapping)
- `pg` - PostgreSQL driver for Node.js

---

## Step 2: Start PostgreSQL (Using Docker)

If you have Docker installed, run:

```bash
docker run --name postgres-nestjs \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=nestjs_db \
  -p 5432:5432 \
  -d postgres:15
```

Or use your system PostgreSQL installation.

Test connection:
```bash
psql -U postgres -h localhost -d nestjs_db
```

---

## Step 3: Configure TypeORM in App Module

### `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksModule } from './tasks/tasks.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'password',
      database: 'nestjs_db',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true, // Auto-create tables (dev only!)
      logging: true, // See SQL queries
    }),
    TasksModule,
  ],
})
export class AppModule {}
```

**⚠️ Important:** `synchronize: true` only for development! Removes tables on restart.

---

## Step 4: Create Task Entity (Database Table)

### `src/tasks/task.entity.ts`

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('tasks')  // Table name in database
export class Task {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column({ default: false })
  completed: boolean;

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;
}
```

**Analogy:** Entity = Table definition
- `@Entity` = "Create table called 'tasks'"
- `@PrimaryGeneratedColumn` = "Auto-increment ID (primary key)"
- `@Column` = "Add a column"

---

## Step 5: Update Tasks Module

### `src/tasks/tasks.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksController } from './tasks.controller';
import { TasksService } from './tasks.service';
import { Task } from './task.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Task])],
  controllers: [TasksController],
  providers: [TasksService],
})
export class TasksModule {}
```

`forFeature([Task])` registers the Task repository for dependency injection.

---

## Step 6: Update Tasks Service (Use Repository)

### `src/tasks/tasks.service.ts`

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Task } from './task.entity';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Injectable()
export class TasksService {
  constructor(
    @InjectRepository(Task)
    private taskRepository: Repository<Task>,
  ) {}

  // Get all tasks
  findAll(): Promise<Task[]> {
    return this.taskRepository.find();
  }

  // Get one task by ID
  async findOne(id: number): Promise<Task> {
    const task = await this.taskRepository.findOne({ where: { id } });
    if (!task) throw new NotFoundException(`Task ${id} not found`);
    return task;
  }

  // Create new task
  create(dto: CreateTaskDto): Promise<Task> {
    const task = this.taskRepository.create(dto);
    return this.taskRepository.save(task);
  }

  // Update task
  async update(id: number, dto: UpdateTaskDto): Promise<Task> {
    const task = await this.findOne(id);
    Object.assign(task, dto);
    return this.taskRepository.save(task);
  }

  // Delete task
  async remove(id: number): Promise<void> {
    const result = await this.taskRepository.delete(id);
    if (result.affected === 0) {
      throw new NotFoundException(`Task ${id} not found`);
    }
  }
}
```

**Key difference from in-memory:**
- `this.taskRepository.find()` = Query database (not array)
- `this.taskRepository.save(task)` = Persist to database
- `this.taskRepository.delete(id)` = Delete from database

---

## Step 7: Test It!

Start your server:
```bash
npm run start:dev
```

Create a task:
```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn TypeORM"}'
```

Get all tasks:
```bash
curl http://localhost:3000/tasks
```

**Check the database:**
```bash
psql -U postgres -h localhost -d nestjs_db
SELECT * FROM tasks;
```

---

## Common Patterns

### Query with Conditions

```typescript
// Find completed tasks
const completedTasks = await this.taskRepository.find({
  where: { completed: true },
});

// Find task with title containing "Learn"
const tasks = await this.taskRepository.find({
  where: { title: Like('%Learn%') },
});
```

### Pagination

```typescript
const page = 1;
const limit = 10;

const tasks = await this.taskRepository.find({
  skip: (page - 1) * limit,
  take: limit,
  order: { createdAt: 'DESC' },
});
```

### Transactions

```typescript
async createMultipleTasks(dtos: CreateTaskDto[]) {
  const queryRunner = this.taskRepository.manager.connection.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    for (const dto of dtos) {
      await queryRunner.manager.save(Task, dto);
    }
    await queryRunner.commitTransaction();
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

---

## Repository Pattern Explanation

**Why use repositories?**

- **Abstraction:** Your service doesn't know about SQL
- **Testability:** Mock repository easily
- **Consistency:** One place for database queries

**Analogy:** Repository = Bank teller
- You don't go directly to vault (database)
- You ask teller (repository) to get/save money
- Teller knows how to interact with vault

---

## Next: Relationships

When you're ready, learn:
- One-to-Many: User has many Tasks
- Many-to-One: Task belongs to User
- Many-to-Many: Tasks shared between multiple Users

---

## Quick Reference

| In-Memory | TypeORM |
|-----------|---------|
| `this.tasks = []` | Table in database |
| `this.tasks.push(task)` | `repository.save(task)` |
| `this.tasks.find(...)` | `repository.find()` |
| `this.tasks.filter(...)` | `repository.find({ where: {...} })` |
| Restart = data lost | Restart = data persists |

---

## Troubleshooting

**"connect ECONNREFUSED 127.0.0.1:5432"**
- PostgreSQL not running. Start it:
  ```bash
  docker start postgres-nestjs
  ```

**"relation \"tasks\" does not exist"**
- Restart with `synchronize: true` to auto-create tables

**"Queries are slow"**
- Add indexes:
  ```typescript
  @Column({ unique: true })
  uniqueField: string;
  ```

---

## Summary

You now have:
- ✅ Real data persistence
- ✅ Database integration
- ✅ Scalable data layer

Next step: **Authentication** - Protect your API from unauthorized access!
