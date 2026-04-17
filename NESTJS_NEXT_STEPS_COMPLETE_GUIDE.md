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

## Step 1.1: Install Prisma Dependencies

```bash
npm install @prisma/client
npm install -D prisma
```

---

## Step 1.2: Initialize Prisma

```bash
npx prisma init
```

This creates:
- `.env` file (for database URL)
- `prisma/schema.prisma` (database schema)

---

## Step 1.3: Configure Database URL

Update `.env`:

```bash
# Replace the DATABASE_URL line with:
DATABASE_URL="postgresql://postgres:password@localhost:5432/nestjs_db"
```

Format: `postgresql://USER:PASSWORD@HOST:PORT/DATABASE`

## Step 1.4: Start PostgreSQL (If Not Running)

**Using Docker:**
```bash
docker run --name postgres-nestjs \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=nestjs_db \
  -p 5432:5432 \
  -d postgres:15
```

**Or restart if already created:**
```bash
docker start postgres-nestjs
```

---

## Step 1.5: Define Database Schema with Prisma

Edit `prisma/schema.prisma`:

### `prisma/schema.prisma` (REPLACE)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Task {
  id        Int     @id @default(autoincrement())
  title     String
  completed Boolean @default(false)
  createdAt DateTime @default(now())
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  password String
}
```

**Prisma schema explanation:**
- `model Task` = Create a table called "tasks"
- `id Int @id @default(autoincrement())` = Auto-increment ID (primary key)
- `@unique` = This field must be unique
- `@default(now())` = Use current timestamp

---

## Step 1.6: Run Migration

Create the database tables:

```bash
npx prisma migrate dev --name init
```

This:
1. Creates migration file in `prisma/migrations/`
2. Runs migration on PostgreSQL
3. Generates Prisma Client

**You'll see:**
```
✔ Generated Prisma Client
✔ Created database tables
Migration complete
```

## Step 1.7: Create Prisma Service (Database Connection)

Create a service that exposes the Prisma client to other services.

### `src/prisma/prisma.service.ts` (NEW FILE)

```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

### `src/prisma/prisma.module.ts` (NEW FILE)

```typescript
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

---

## Step 1.8: Update App Module

### `src/app.module.ts` (REPLACE - SIMPLIFIED!)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TasksModule } from './tasks/tasks.module';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
    }),
    PrismaModule,
    TasksModule,
  ],
})
export class AppModule {}
```

Much simpler! Prisma handles the database connection, no need for TypeOrmModule.

---

## Step 1.9: Update Tasks Service (Use Prisma)

Replace your in-memory service with Prisma queries.

### `src/tasks/tasks.service.ts` (REPLACE EVERYTHING)

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Injectable()
export class TasksService {
  constructor(private prisma: PrismaService) {}

  // GET all tasks
  findAll() {
    return this.prisma.task.findMany({
      orderBy: { createdAt: 'desc' },
    });
  }

  // GET one task by ID
  async findOne(id: number) {
    const task = await this.prisma.task.findUnique({
      where: { id },
    });

    if (!task) {
      throw new NotFoundException(`Task ${id} not found`);
    }

    return task;
  }

  // POST create new task
  create(dto: CreateTaskDto) {
    return this.prisma.task.create({
      data: dto,
    });
  }

  // PATCH update task
  async update(id: number, dto: UpdateTaskDto) {
    await this.findOne(id); // Verify exists

    return this.prisma.task.update({
      where: { id },
      data: dto,
    });
  }

  // DELETE task
  async remove(id: number) {
    await this.findOne(id); // Verify exists

    await this.prisma.task.delete({
      where: { id },
    });
  }
}
```

**Prisma syntax:**
- `prisma.task.findMany()` = Get all tasks
- `prisma.task.findUnique()` = Get one by unique field
- `prisma.task.create()` = Create new
- `prisma.task.update()` = Update existing
- `prisma.task.delete()` = Delete

---

## Step 1.10: Update Tasks Module

### `src/tasks/tasks.module.ts` (REPLACE)

```typescript
import { Module } from '@nestjs/common';
import { TasksController } from './tasks.controller';
import { TasksService } from './tasks.service';
import { PrismaModule } from '../prisma/prisma.module';

@Module({
  imports: [PrismaModule],
  controllers: [TasksController],
  providers: [TasksService],
})
export class TasksModule {}
```

---

## Step 1.11: Test Database Integration

Start your server:
```bash
npm run start:dev
```

**Create a task:**
```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Prisma"}'
```

**Get all tasks:**
```bash
curl http://localhost:3000/tasks
```

**Check database directly:**
```bash
npx prisma studio
```

This opens a GUI to view/edit database data!

---

## ✅ Part 1 Complete

Your API now:
- Stores tasks in PostgreSQL via Prisma
- Survives app restart
- Can query data easily
- Uses type-safe Prisma queries

---

# PART 2: CONFIGURATION

## The Problem

Right now, your database URL is in `.env`:
```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/nestjs_db"
```

**Problems:**
1. Same DATABASE_URL for dev and production (doesn't work)
2. Can't change config without restarting app
3. Secret password might leak

**Solution:** Use .env files with @nestjs/config

---

## Step 2.1: Install Config Package

```bash
npm install @nestjs/config
```

---

## Step 2.2: Update `.env` File

You already have `.env` from Prisma init. Update it:

### `.env` (MODIFY)

```bash
# Database (Prisma)
DATABASE_URL="postgresql://postgres:password@localhost:5432/nestjs_db"

# JWT (we'll use in Part 3)
JWT_SECRET=your-super-long-random-secret-change-this-in-production
JWT_EXPIRATION=1h

# App
APP_PORT=3000
NODE_ENV=development
```

---

## Step 2.3: Update `.gitignore`

Make sure `.env` is NOT committed:

### `.gitignore` (VERIFY - SHOULD ALREADY HAVE)

```bash
.env
.env*.local
.prisma/
```

Prisma init already adds these, but verify!

---

## Step 2.4: Create Config Files

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
import { TasksModule } from './tasks/tasks.module';
import { PrismaModule } from './prisma/prisma.module';
import appConfig from './config/app.config';
import jwtConfig from './config/jwt.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      load: [appConfig, jwtConfig],
    }),
    PrismaModule,
    TasksModule,
  ],
})
export class AppModule {}
```

---

## Step 2.6: Update `main.ts` (Use Config for Port)

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
```

**Change config without restarting:**
1. Edit `.env`: change `APP_PORT=3000` to `APP_PORT=3001`
2. Server restarts automatically
3. Run: `curl http://localhost:3001/tasks`

---

## ✅ Part 2 Complete

Your API now:
- Reads config from `.env`
- Can have different `.env` files per environment
- Secrets not in code
- Port and other settings are configurable

---

# PART 3: AUTHENTICATION

## The Problem

Right now, **anyone can access your API**:
```bash
curl http://localhost:3000/tasks  # Works for anyone!
```

**Solution:** Users must log in and get a JWT token.

---

## Step 3.1: Install Auth Dependencies

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcryptjs
npm install -D @types/passport-jwt @types/bcryptjs
```

## Step 3.2: Update Prisma Schema (Add User Model)

Update `prisma/schema.prisma`:

### `prisma/schema.prisma` (MODIFY - ADD USER MODEL)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Task {
  id        Int     @id @default(autoincrement())
  title     String
  completed Boolean @default(false)
  createdAt DateTime @default(now())
}

model User {
  id       Int     @id @default(autoincrement())
  email    String  @unique
  password String
}
```

---

## Step 3.3: Run Migration

Create the users table:

```bash
npx prisma migrate dev --name add_users
```

---

## Step 3.4: Create Users Service

### `src/users/users.service.ts` (NEW FILE)

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import * as bcrypt from 'bcryptjs';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  // Find user by email
  findByEmail(email: string) {
    return this.prisma.user.findUnique({
      where: { email },
    });
  }

  // Create new user with hashed password
  async create(email: string, password: string) {
    const hashedPassword = await bcrypt.hash(password, 10);
    
    return this.prisma.user.create({
      data: {
        email,
        password: hashedPassword,
      },
    });
  }

  // Compare password with hash
  async validatePassword(user: any, password: string): Promise<boolean> {
    return bcrypt.compare(password, user.password);
  }
}
```

### `src/users/users.module.ts` (NEW FILE)

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { PrismaModule } from '../prisma/prisma.module';

@Module({
  imports: [PrismaModule],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

---

## Step 3.5: Create Auth Module

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
    const existingUser = await this.usersService.findByEmail(registerDto.email);
    if (existingUser) {
      throw new UnauthorizedException('User already exists');
    }

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
    const user = await this.usersService.findByEmail(loginDto.email);
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await this.usersService.validatePassword(
      user,
      loginDto.password,
    );
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const payload = { sub: user.id, email: user.email };

    return {
      access_token: this.jwtService.sign(payload),
    };
  }

  // Validate JWT token
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

### `src/auth/auth.controller.ts` (NEW FILE)

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

  @UseGuards(JwtGuard)
  @Get('me')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

### Auth Module

### `src/auth/auth.module.ts` (NEW FILE)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
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

## Step 3.6: Update App Module (Add Auth & Users)

### `src/app.module.ts` (MODIFY)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TasksModule } from './tasks/tasks.module';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';
import appConfig from './config/app.config';
import jwtConfig from './config/jwt.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      load: [appConfig, jwtConfig],
    }),
    PrismaModule,
    AuthModule,
    UsersModule,
    TasksModule,
  ],
})
export class AppModule {}
```

---

## Step 3.7: Protect Tasks Routes

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
