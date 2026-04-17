# NestJS Next Steps: From In-Memory to Production-Ready API

**Goal:** Build on your in-memory Tasks API by adding database, configuration management, and authentication—all in one guided journey.

**What you'll build:**
- ✅ In-memory tasks API (you already have this)
- ➕ Real database persistence (PostgreSQL + TypeORM)
- ➕ Environment configuration (dev/test/prod)
- ➕ User authentication (JWT + password hashing)

**Result:** A complete, production-ready API where users log in and manage their tasks in a real database.

---

## Table of Contents

1. [Part 1: Add Database (TypeORM + PostgreSQL)](#part-1-database)
2. [Part 2: Add Configuration Management (.env files)](#part-2-configuration)
3. [Part 3: Add Authentication (JWT + Users)](#part-3-authentication)
4. [Testing Everything Together](#testing-everything)

---

# PART 1: DATABASE

## The Problem

Your current API stores tasks in memory:
```typescript
private tasks = [];  // Disappears when app restarts!
```

**Solution:** Store data in PostgreSQL (a real database).

---

## Step 1.1: Install Dependencies

```bash
npm install @nestjs/typeorm typeorm pg
```

---

## Step 1.2: Start PostgreSQL

**Option A: Using Docker (recommended)**
```bash
docker run --name postgres-nestjs \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=nestjs_db \
  -p 5432:5432 \
  -d postgres:15
```

**Option B: Using local PostgreSQL**
- Install from postgresql.org
- Create database: `createdb nestjs_db`

**Test connection:**
```bash
psql -U postgres -h localhost -d nestjs_db
# Type \q to exit
```

---

## Step 1.3: Create Task Entity

The Task entity is like a **table definition** in the database.

### `src/tasks/task.entity.ts` (NEW FILE)

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('tasks')  // Table name = 'tasks'
export class Task {
  @PrimaryGeneratedColumn()
  id: number;  // Auto-increment primary key

  @Column()
  title: string;

  @Column({ default: false })
  completed: boolean;

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;
}
```

**Analogy:**
- `@Entity('tasks')` = "Create a table called tasks"
- `@PrimaryGeneratedColumn()` = "Auto-increment ID"
- `@Column()` = "Add a column to the table"

---

## Step 1.4: Configure TypeORM in App Module

Update your app.module.ts:

### `src/app.module.ts` (MODIFY)

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksModule } from './tasks/tasks.module';
import { Task } from './tasks/task.entity';

@Module({
  imports: [
    // TypeORM Configuration
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'password',
      database: 'nestjs_db',
      entities: [Task],  // Register entities here
      synchronize: true, // Auto-create tables (dev only!)
      logging: true,     // See SQL queries in console
    }),
    TasksModule,
  ],
})
export class AppModule {}
```

⚠️ **Important:** `synchronize: true` only for development! We'll change this later.

---

## Step 1.5: Update Tasks Module

Register the Task entity as a repository so the service can use it.

### `src/tasks/tasks.module.ts` (MODIFY)

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksController } from './tasks.controller';
import { TasksService } from './tasks.service';
import { Task } from './task.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Task])],  // NEW: Register Task repository
  controllers: [TasksController],
  providers: [TasksService],
})
export class TasksModule {}
```

---

## Step 1.6: Update Tasks Service (Use Repository)

Replace your in-memory array with database queries.

### `src/tasks/tasks.service.ts` (REPLACE EVERYTHING)

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

  // GET all tasks
  findAll(): Promise<Task[]> {
    return this.taskRepository.find({
      order: { createdAt: 'DESC' },
    });
  }

  // GET one task by ID
  async findOne(id: number): Promise<Task> {
    const task = await this.taskRepository.findOne({ where: { id } });
    if (!task) {
      throw new NotFoundException(`Task ${id} not found`);
    }
    return task;
  }

  // POST create new task
  create(dto: CreateTaskDto): Promise<Task> {
    const task = this.taskRepository.create(dto);
    return this.taskRepository.save(task);
  }

  // PATCH update task
  async update(id: number, dto: UpdateTaskDto): Promise<Task> {
    const task = await this.findOne(id);
    Object.assign(task, dto);
    return this.taskRepository.save(task);
  }

  // DELETE task
  async remove(id: number): Promise<void> {
    const result = await this.taskRepository.delete(id);
    if (result.affected === 0) {
      throw new NotFoundException(`Task ${id} not found`);
    }
  }
}
```

**Key differences from in-memory:**
- `this.taskRepository.find()` = Query database
- `this.taskRepository.save(task)` = Persist to database
- `this.taskRepository.delete(id)` = Remove from database
- All methods return `Promise` (database queries are async)

---

## Step 1.7: Test Database Integration

Start your server:
```bash
npm run start:dev
```

**Create a task:**
```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Database"}'
```

**Get all tasks:**
```bash
curl http://localhost:3000/tasks
```

**Check database directly:**
```bash
psql -U postgres -h localhost -d nestjs_db
SELECT * FROM tasks;
```

**Expected output:** Task appears in both curl response AND database!

---

## ✅ Part 1 Complete

Your API now:
- Stores tasks in a real database
- Survives app restart
- Can be queried from multiple clients
- Shows SQL queries in console (because `logging: true`)

**Problems solved:**
- ✅ Data persistence
- ❌ Secret password in code (we'll fix next)
- ❌ Can't have separate dev/prod databases (we'll fix next)
- ❌ Anyone can access the API (we'll fix later)

---

# PART 2: CONFIGURATION

## The Problem

Right now, your database config is hard-coded:
```typescript
host: 'localhost',
password: 'password',
database: 'nestjs_db',
```

**Problems:**
1. Secret password in source code (security breach!)
2. Same config for dev and production (doesn't work)
3. Can't change config without rebuilding code

**Solution:** Use `.env` files (not committed to git).

---

## Step 2.1: Install Config Package

```bash
npm install @nestjs/config
```

---

## Step 2.2: Create `.env` File

In your project root (same folder as package.json):

### `.env` (NEW FILE - DO NOT COMMIT TO GIT!)

```bash
# Database Configuration
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=password
DATABASE_NAME=nestjs_db

# JWT (we'll use this in Part 3)
JWT_SECRET=your-super-long-random-secret-change-this-in-production
JWT_EXPIRATION=1h

# App
APP_PORT=3000
NODE_ENV=development
```

---

## Step 2.3: Update `.gitignore`

Make sure `.env` is NOT committed:

### `.gitignore` (MODIFY - ADD THIS LINE)

```bash
.env
.env.local
```

---

## Step 2.4: Create Config Files

Organize config in separate, reusable files:

### `src/config/database.config.ts` (NEW FILE)

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT, 10) || 5432,
  username: process.env.DATABASE_USERNAME,
  password: process.env.DATABASE_PASSWORD,
  database: process.env.DATABASE_NAME,
}));
```

### `src/config/app.config.ts` (NEW FILE)

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  port: parseInt(process.env.APP_PORT, 10) || 3000,
  environment: process.env.NODE_ENV || 'development',
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
}));
```

### `src/config/jwt.config.ts` (NEW FILE)

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  expiresIn: process.env.JWT_EXPIRATION || '1h',
}));
```

---

## Step 2.5: Update App Module (Load Config)

### `src/app.module.ts` (MODIFY)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksModule } from './tasks/tasks.module';
import { Task } from './tasks/task.entity';
import databaseConfig from './config/database.config';
import appConfig from './config/app.config';
import jwtConfig from './config/jwt.config';

@Module({
  imports: [
    // Load environment variables from .env
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,  // Available everywhere without importing
      load: [databaseConfig, appConfig, jwtConfig],
    }),

    // TypeORM Configuration (now uses ConfigService)
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        const dbConfig = configService.get('database');
        return {
          type: 'postgres',
          host: dbConfig.host,
          port: dbConfig.port,
          username: dbConfig.username,
          password: dbConfig.password,
          database: dbConfig.database,
          entities: [Task],
          synchronize: configService.get('app.isDevelopment'),
          logging: configService.get('app.isDevelopment'),
        };
      },
    }),

    TasksModule,
  ],
})
export class AppModule {}
```

**Key changes:**
- `ConfigModule.forRoot()` loads `.env`
- `TypeOrmModule.forRootAsync()` waits for config to load
- Database config comes from `configService.get('database')`

---

## Step 2.6: Update `main.ts` (Use App Port from Config)

### `src/main.ts` (MODIFY)

```typescript
import { NestFactory } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  
  const port = configService.get('app.port');
  const environment = configService.get('app.environment');

  await app.listen(port);
  console.log(`🚀 Application running on port ${port} (${environment})`);
}

bootstrap();
```

---

## Step 2.7: Test Configuration

Start your server:
```bash
npm run start:dev
```

**You should see:**
```
🚀 Application running on port 3000 (development)
[TypeOrmModule] info  ...  // SQL queries logged because isDevelopment=true
```

**Change config without restarting:**
1. Edit `.env`: change `APP_PORT=3000` to `APP_PORT=3001`
2. The app should restart automatically (because `npm run start:dev` watches files)
3. It should now run on port 3001

**Verify new port:**
```bash
curl http://localhost:3001/tasks
```

---

## ✅ Part 2 Complete

Your API now:
- Reads config from `.env` (not in code)
- Can have separate `.env.development`, `.env.test`, `.env.production`
- Doesn't expose secrets in source code
- Can change config without rebuilding

---

# PART 3: AUTHENTICATION

## The Problem

Right now, **anyone can access your API**:
```bash
curl http://localhost:3000/tasks  # Works for anyone!
```

**Solution:** Users must log in and get a JWT token to access tasks.

---

## Step 3.1: Install Auth Dependencies

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcryptjs
npm install -D @types/passport-jwt @types/bcryptjs
```

---

## Step 3.2: Create User Entity

Users need to be stored in the database, just like tasks.

### `src/users/user.entity.ts` (NEW FILE)

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;  // Will be hashed, not stored as plain text
}
```

---

## Step 3.3: Create Users Module & Service

### Generate files:
```bash
nest g module users --no-spec
nest g service users --no-spec
```

### `src/users/users.service.ts` (REPLACE)

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import * as bcrypt from 'bcryptjs';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  // Find user by email
  findByEmail(email: string): Promise<User | null> {
    return this.usersRepository.findOne({ where: { email } });
  }

  // Create new user with hashed password
  async create(email: string, password: string): Promise<User> {
    // Hash password (don't store plain text!)
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = this.usersRepository.create({
      email,
      password: hashedPassword,
    });
    return this.usersRepository.save(user);
  }

  // Compare password with hash
  async validatePassword(user: User, password: string): Promise<boolean> {
    return bcrypt.compare(password, user.password);
  }
}
```

### `src/users/users.module.ts` (REPLACE)

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  exports: [UsersService],  // Export so AuthModule can use it
})
export class UsersModule {}
```

---

## Step 3.4: Create Auth Module

### Generate files:
```bash
nest g module auth --no-spec
nest g controller auth --no-spec
nest g service auth --no-spec
```

### Create DTOs

### `src/auth/dto/login.dto.ts` (NEW FILE)

```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

### `src/auth/dto/register.dto.ts` (NEW FILE)

```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

### Auth Service

### `src/auth/auth.service.ts` (NEW FILE)

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { UsersService } from '../users/users.service';
import { LoginDto } from './dto/login.dto';
import { RegisterDto } from './dto/register.dto';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  // Register new user
  async register(registerDto: RegisterDto) {
    // Check if user already exists
    const existingUser = await this.usersService.findByEmail(registerDto.email);
    if (existingUser) {
      throw new UnauthorizedException('User already exists');
    }

    // Create new user
    const user = await this.usersService.create(
      registerDto.email,
      registerDto.password,
    );

    return {
      id: user.id,
      email: user.email,
    };
  }

  // Login user and return JWT token
  async login(loginDto: LoginDto) {
    // Find user by email
    const user = await this.usersService.findByEmail(loginDto.email);
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    // Verify password
    const isPasswordValid = await this.usersService.validatePassword(
      user,
      loginDto.password,
    );
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    // Create JWT payload
    const payload = { sub: user.id, email: user.email };

    // Sign and return token
    return {
      access_token: this.jwtService.sign(payload),
    };
  }

  // Validate JWT token (used by strategy)
  async validateToken(payload: any) {
    const user = await this.usersService.findByEmail(payload.email);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }
    return user;
  }
}
```

### JWT Strategy

### `src/auth/strategies/jwt.strategy.ts` (NEW FILE)

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { AuthService } from '../auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private authService: AuthService,
    private configService: ConfigService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('jwt.secret'),
    });
  }

  async validate(payload: any) {
    return this.authService.validateToken(payload);
  }
}
```

### JWT Guard

### `src/auth/guards/jwt.guard.ts` (NEW FILE)

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtGuard extends AuthGuard('jwt') {}
```

### Auth Controller

### `src/auth/auth.controller.ts` (REPLACE)

```typescript
import { Controller, Post, Body, UseGuards, Get, Request } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';
import { RegisterDto } from './dto/register.dto';
import { JwtGuard } from './guards/jwt.guard';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }

  @Post('login')
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  // Protected route - only authenticated users can access
  @UseGuards(JwtGuard)
  @Get('me')
  getProfile(@Request() req) {
    return req.user;  // req.user is set by JwtGuard
  }
}
```

### Auth Module

### `src/auth/auth.module.ts` (REPLACE)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';
import { User } from '../users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    UsersModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get('jwt.secret'),
        signOptions: {
          expiresIn: configService.get('jwt.expiresIn'),
        },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
  exports: [AuthService, JwtStrategy],
})
export class AuthModule {}
```

---

## Step 3.5: Update App Module (Add Auth & Users)

### `src/app.module.ts` (MODIFY - ADD IMPORTS)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TasksModule } from './tasks/tasks.module';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';
import { Task } from './tasks/task.entity';
import { User } from './users/user.entity';
import databaseConfig from './config/database.config';
import appConfig from './config/app.config';
import jwtConfig from './config/jwt.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      load: [databaseConfig, appConfig, jwtConfig],
    }),

    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        const dbConfig = configService.get('database');
        return {
          type: 'postgres',
          host: dbConfig.host,
          port: dbConfig.port,
          username: dbConfig.username,
          password: dbConfig.password,
          database: dbConfig.database,
          entities: [Task, User],  // ADD User entity
          synchronize: configService.get('app.isDevelopment'),
          logging: configService.get('app.isDevelopment'),
        };
      },
    }),

    AuthModule,   // ADD
    UsersModule,  // ADD
    TasksModule,
  ],
})
export class AppModule {}
```

---

## Step 3.6: Protect Tasks Routes with Authentication

Now only authenticated users can access tasks!

### `src/tasks/tasks.controller.ts` (MODIFY)

```typescript
import {
  Controller,
  Get,
  Post,
  Patch,
  Delete,
  Param,
  Body,
  UseGuards,
  Request,
  ParseIntPipe,
} from '@nestjs/common';
import { JwtGuard } from '../auth/guards/jwt.guard';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Controller('tasks')
@UseGuards(JwtGuard)  // ALL routes require authentication
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Get()
  findAll(@Request() req) {
    console.log(`Tasks requested by: ${req.user.email}`);
    return this.tasksService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.tasksService.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateTaskDto, @Request() req) {
    console.log(`Task created by: ${req.user.email}`);
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

# TESTING EVERYTHING

Now your complete API is ready! Let's test it end-to-end.

## Start Server

```bash
npm run start:dev
```

You should see:
```
🚀 Application running on port 3000 (development)
[TypeOrmModule] Database connected
```

---

## Step 1: Register a User

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'
```

**Response:**
```json
{
  "id": 1,
  "email": "user@example.com"
}
```

---

## Step 2: Try to Access Tasks Without Token (Should Fail)

```bash
curl http://localhost:3000/tasks
```

**Response:**
```json
{
  "message": "Unauthorized",
  "statusCode": 401
}
```

✅ Good! Tasks are protected.

---

## Step 3: Login to Get Token

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsImVtYWlsIjoidXNlckBleGFtcGxlLmNvbSIsImlhdCI6MTYzNTQwNzY0MCwiZXhwIjoxNjM1NDExMjQwfQ...."
}
```

**Save the token (without quotes):**
```bash
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsImVtYWlsIjoidXNlckBleGFtcGxlLmNvbSIsImlhdCI6MTYzNTQwNzY0MCwiZXhwIjoxNjM1NDExMjQwfQ...."
```

---

## Step 4: Access Protected Route with Token

```bash
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "id": 1,
  "email": "user@example.com"
}
```

✅ Token works!

---

## Step 5: Create Tasks (With Token)

```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Learn NestJS","completed":false}'
```

**Response:**
```json
{
  "id": 1,
  "title": "Learn NestJS",
  "completed": false,
  "createdAt": "2024-01-15T10:30:00.000Z"
}
```

---

## Step 6: Get Tasks (With Token)

```bash
curl http://localhost:3000/tasks \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
[
  {
    "id": 1,
    "title": "Learn NestJS",
    "completed": false,
    "createdAt": "2024-01-15T10:30:00.000Z"
  }
]
```

✅ Stored in database!

---

## Step 7: Update Task

```bash
curl -X PATCH http://localhost:3000/tasks/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"completed":true}'
```

---

## Step 8: Delete Task

```bash
curl -X DELETE http://localhost:3000/tasks/1 \
  -H "Authorization: Bearer $TOKEN"
```

---

## Verify in Database

```bash
psql -U postgres -h localhost -d nestjs_db
SELECT * FROM users;
SELECT * FROM tasks;
```

You should see:
- 1 user (user@example.com)
- Tasks in the database

---

# 🎉 YOU'RE DONE!

## What You Built

Your API now has:

✅ **Database Persistence**
- Tasks stored in PostgreSQL
- Survives app restart

✅ **Configuration Management**
- Environment variables in `.env`
- Different configs for dev/prod
- Secrets not in code

✅ **Authentication**
- User registration
- Login with password hashing
- JWT tokens
- Protected routes

✅ **Architecture**
- Controller → Service → Repository pattern
- Dependency injection
- Type-safe TypeScript

---

## Architecture Diagram

```
HTTP Request
    ↓
Controller (route handler)
    ↓
JwtGuard (validates token)
    ↓
req.user (authenticated user)
    ↓
Service (business logic)
    ↓
Repository (database queries)
    ↓
PostgreSQL Database
```

---

## Next Topics to Explore

1. **Relationships** - Link users to tasks (each user's own tasks)
2. **Pagination** - Get tasks in pages (10 per page)
3. **Filtering** - Get only completed tasks
4. **Testing** - Write tests for your API
5. **Validation Pipes** - Better error messages
6. **Exception Filters** - Consistent error format
7. **Interceptors** - Log all requests
8. **Caching** - Make API faster
9. **Rate Limiting** - Protect from abuse
10. **Deployment** - Deploy to production!

---

## Quick Reference

### Database
```bash
psql -U postgres -h localhost -d nestjs_db
SELECT * FROM tasks;
SELECT * FROM users;
```

### Stop PostgreSQL
```bash
docker stop postgres-nestjs
```

### Restart PostgreSQL
```bash
docker start postgres-nestjs
```

### Environment Variables
Edit `.env` and restart server with `npm run start:dev`

### Generate NestJS Files
```bash
nest g controller name
nest g service name
nest g module name
```

---

## Common Issues

**"connect ECONNREFUSED"**
- PostgreSQL not running. Start with: `docker start postgres-nestjs`

**"Unauthorized" on protected routes**
- Missing token in header
- Check format: `Authorization: Bearer TOKEN`
- Token expired (1 hour default)

**"relation \"users\" does not exist"**
- Restart with `synchronize: true` to auto-create tables

**"User already exists"**
- Register with a different email address

---

## Security Reminders

✅ Never commit `.env` to git  
✅ Keep JWT_SECRET long and random  
✅ Change JWT_SECRET in production  
✅ Use HTTPS in production  
✅ Don't log passwords or tokens  
✅ Rotate secrets regularly  

---

## Summary

You now have a **production-ready NestJS API**:
- Real database with persistent data
- Configuration management for different environments
- User authentication with JWT tokens
- Protected routes

🚀 **Ready to deploy!**
