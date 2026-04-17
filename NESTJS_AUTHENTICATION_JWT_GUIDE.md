# NestJS Authentication with JWT - Beginner's Guide

After persisting data in a database, you need to **protect your API**. This guide teaches JWT authentication step-by-step.

---

## What You'll Learn

- Why authentication matters
- JWT (JSON Web Tokens) concept
- Creating login endpoint
- Protecting routes with Guards
- Extracting user from token

---

## Authentication Concept (With Analogy)

**Without authentication:**
- Anyone can access your API
- Like a hotel where anyone can enter any room

**With authentication (password login):**
- User proves identity with username/password
- Like showing ID at hotel reception

**With JWT (token-based):**
- After login, user gets a "key card" (token)
- Every future request, user shows the key card
- Server validates the key card is real

---

## Why JWT?

**Problem with passwords:**
- Server stores password (security risk if hacked)
- Can't verify multiple servers (distributed systems)

**Solution with JWT:**
- No password sent after login
- Token is cryptographically signed
- Each server can verify token independently
- Stateless (server doesn't store sessions)

---

## Step 1: Install Dependencies

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcryptjs
npm install -D @types/passport-jwt @types/bcryptjs
```

**What each does:**
- `@nestjs/jwt` - Sign and verify JWT tokens
- `@nestjs/passport` - Passport.js integration
- `passport-jwt` - Strategy for JWT validation
- `bcryptjs` - Hash passwords securely

---

## Step 2: Create Users Module

Generate files:
```bash
nest g module users
nest g service users --no-spec
```

### `src/users/user.entity.ts`

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string; // Will be hashed
}
```

### `src/users/users.service.ts`

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
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = this.usersRepository.create({
      email,
      password: hashedPassword,
    });
    return this.usersRepository.save(user);
  }

  // Verify password
  async validatePassword(user: User, password: string): Promise<boolean> {
    return bcrypt.compare(password, user.password);
  }
}
```

### `src/users/users.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  exports: [UsersService], // Export so AuthModule can use it
})
export class UsersModule {}
```

---

## Step 3: Create Auth Module

Generate files:
```bash
nest g module auth
nest g controller auth --no-spec
nest g service auth --no-spec
```

### `src/auth/dto/login.dto.ts`

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

### `src/auth/auth.service.ts`

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

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

    // Create JWT payload
    const payload = { sub: user.id, email: user.email };

    // Sign and return token
    return {
      access_token: this.jwtService.sign(payload, {
        expiresIn: '1h',
      }),
    };
  }

  // Validate JWT token (used by strategy)
  async validateToken(payload: any) {
    // payload = { sub: userId, email: userEmail, iat, exp }
    const user = await this.usersService.findByEmail(payload.email);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }
    return user;
  }
}
```

### `src/auth/jwt.strategy.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { AuthService } from './auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: 'your-secret-key-change-me', // Move to env!
    });
  }

  async validate(payload: any) {
    return this.authService.validateToken(payload);
  }
}
```

### `src/auth/jwt.guard.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtGuard extends AuthGuard('jwt') {}
```

### `src/auth/auth.controller.ts`

```typescript
import { Controller, Post, Body, UseGuards, Get, Request } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';
import { JwtGuard } from './jwt.guard';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  // Protected route (requires valid JWT)
  @UseGuards(JwtGuard)
  @Get('me')
  getProfile(@Request() req) {
    return req.user; // req.user is set by JwtGuard
  }
}
```

### `src/auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: 'your-secret-key-change-me', // Move to .env!
      signOptions: { expiresIn: '1h' },
    }),
  ],
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
})
export class AuthModule {}
```

### Update `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';

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
      synchronize: true,
    }),
    AuthModule,
    UsersModule,
  ],
})
export class AppModule {}
```

---

## Step 4: Protect Your Existing Routes

### Update `src/tasks/tasks.controller.ts`

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
import { JwtGuard } from '../auth/jwt.guard';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';

@Controller('tasks')
@UseGuards(JwtGuard) // All routes require JWT
export class TasksController {
  constructor(private readonly taskService: TasksService) {}

  @Get()
  findAll(@Request() req) {
    // req.user contains authenticated user
    console.log('Tasks requested by:', req.user.email);
    return this.taskService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.taskService.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateTaskDto, @Request() req) {
    console.log('Task created by:', req.user.email);
    return this.taskService.create(dto);
  }

  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() dto: UpdateTaskDto,
  ) {
    return this.taskService.update(id, dto);
  }

  @Delete(':id')
  delete(@Param('id', ParseIntPipe) id: number) {
    this.taskService.remove(id);
    return { message: `Task ${id} deleted` };
  }
}
```

---

## Step 5: Test It!

### 1) Create a user (or use SQL directly)

```bash
# Via SQL:
psql -U postgres -h localhost -d nestjs_db
INSERT INTO users (email, password) VALUES ('user@example.com', 'hashed_pwd');
```

### 2) Login to get token

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'

# Response:
# {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

### 3) Use token to access protected route

```bash
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Response:
# {"id":1,"email":"user@example.com"}
```

### 4) Try to access tasks without token

```bash
curl http://localhost:3000/tasks
# Response: {"message":"Unauthorized","statusCode":401}
```

### 5) Access tasks with token

```bash
curl http://localhost:3000/tasks \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
# Response: Array of tasks
```

---

## Flow Diagram

```
1. POST /auth/login {"email", "password"}
   ↓
2. AuthService.login()
   - Verify email exists
   - Verify password matches
   - Create JWT token
   ↓
3. Return {"access_token": "..."}

---

4. GET /tasks
   + Header: Authorization: Bearer TOKEN
   ↓
5. JwtGuard validates token
   - Extract token from header
   - Verify signature (using secret)
   - Verify not expired
   ↓
6. If valid → req.user populated
   If invalid → 401 Unauthorized
   ↓
7. TasksController executes
```

---

## Security Best Practices

### 1) Move secret to environment variable

**`.env`:**
```
JWT_SECRET=very-long-random-secret-keep-it-safe
JWT_EXPIRATION=1h
```

**`auth.module.ts`:**
```typescript
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get('JWT_SECRET'),
        signOptions: {
          expiresIn: configService.get('JWT_EXPIRATION'),
        },
      }),
    }),
  ],
})
export class AuthModule {}
```

### 2) Use HTTPS in production

Tokens should only be sent over HTTPS.

### 3) Refresh tokens

- Short-lived access tokens (1 hour)
- Long-lived refresh tokens (7 days)
- User refreshes token by calling `/auth/refresh`

### 4) Never expose secret

- Use environment variables
- Use secret management service (AWS Secrets Manager, HashiCorp Vault)

---

## Common Patterns

### Optional Authentication

```typescript
@UseGuards(new JwtGuard({ passthrough: true }))
@Get('public-posts')
getPosts(@Request() req) {
  if (req.user) {
    // User authenticated
  } else {
    // User not authenticated (but allowed)
  }
}
```

### Role-Based Access Control (RBAC)

```typescript
// Add role to User entity
@Column()
role: 'admin' | 'user';

// Create role guard
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return request.user.role === 'admin';
  }
}

// Use it
@UseGuards(JwtGuard, RolesGuard)
@Delete(':id')
deleteTask(@Param('id') id: number) {
  // Only admins can delete
}
```

---

## Troubleshooting

**"Unauthorized" on protected routes**
- Check token format: `Authorization: Bearer TOKEN`
- Verify token not expired: decode at jwt.io
- Check JWT_SECRET matches in all places

**"secretOrKey must have a value"**
- Ensure JWT_SECRET is set in env or hardcoded

**"User not found"**
- User deleted after token issued
- Use refresh tokens for resilience

---

## Summary

You now have:
- ✅ User registration/login
- ✅ JWT token generation
- ✅ Protected routes
- ✅ User identification per request

Next step: **Configuration Management** - Use environment variables properly!
