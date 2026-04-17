# NestJS Configuration Management - Beginner's Guide

You've built an API with database and authentication. Now you need to **manage environment-specific settings** securely. This guide teaches configuration best practices.

---

## What You'll Learn

- Environment variables and why they matter
- `@nestjs/config` package
- `.env` files and how to use them
- Different configs per environment (dev, test, prod)
- Secrets management

---

## Why Configuration Matters (With Analogy)

**Hard-coded approach (BAD):**
```typescript
const dbPassword = 'my-super-secret-password'; // EXPOSED IN CODE!
const dbHost = 'localhost'; // Can't change without redeploying
```

**Problem:**
- Secret in source code = everyone sees it (security breach)
- Different hosts for dev vs prod = need to rebuild code
- Like writing passwords on sticky notes on your desk

**Configuration approach (GOOD):**
```bash
# .env file (not in git)
DATABASE_PASSWORD=secret-only-here
DATABASE_HOST=localhost

# app.module.ts
const dbPassword = process.env.DATABASE_PASSWORD;
const dbHost = process.env.DATABASE_HOST;
```

**Benefit:**
- Secrets not in code
- Change config without rebuilding
- Different config per deployment
- Like storing passwords in a locked safe, not on desk

---

## Step 1: Install Configuration Package

```bash
npm install @nestjs/config
```

---

## Step 2: Create `.env` File

In your project root, create `.env` (NOT committed to git):

```bash
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=password
DATABASE_NAME=nestjs_db

# JWT
JWT_SECRET=your-super-long-secret-key-change-this
JWT_EXPIRATION=1h

# App
APP_PORT=3000
NODE_ENV=development
```

**Add to `.gitignore`:**
```
.env
.env.local
.env.*.local
```

---

## Step 3: Configure ConfigModule

### `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';
import { TasksModule } from './tasks/tasks.module';

@Module({
  imports: [
    // Load .env file
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true, // Makes ConfigService available everywhere
      cache: true, // Cache env vars
    }),

    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('DATABASE_HOST'),
        port: configService.get('DATABASE_PORT'),
        username: configService.get('DATABASE_USERNAME'),
        password: configService.get('DATABASE_PASSWORD'),
        database: configService.get('DATABASE_NAME'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: configService.get('NODE_ENV') === 'development',
        logging: configService.get('NODE_ENV') === 'development',
      }),
    }),

    AuthModule,
    UsersModule,
    TasksModule,
  ],
})
export class AppModule {}
```

**Why `forRootAsync`?**
- Loads config **after** `.env` is read
- Synchronous `forRoot` runs before env vars load

---

## Step 4: Use ConfigService in Auth Module

### `src/auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
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
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
})
export class AuthModule {}
```

### `src/auth/jwt.strategy.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { AuthService } from './auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private authService: AuthService,
    private configService: ConfigService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    return this.authService.validateToken(payload);
  }
}
```

---

## Step 5: Create Configuration Files

For more advanced config, create separate files:

### `src/config/database.config.ts`

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

### `src/config/jwt.config.ts`

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  expiresIn: process.env.JWT_EXPIRATION || '1h',
}));
```

### `src/config/app.config.ts`

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  port: parseInt(process.env.APP_PORT, 10) || 3000,
  environment: process.env.NODE_ENV || 'development',
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
}));
```

### Update `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import databaseConfig from './config/database.config';
import jwtConfig from './config/jwt.config';
import appConfig from './config/app.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      load: [databaseConfig, jwtConfig, appConfig], // Load config files
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
          entities: [__dirname + '/**/*.entity{.ts,.js}'],
          synchronize: configService.get('app.isDevelopment'),
        };
      },
    }),

    AuthModule,
    UsersModule,
    TasksModule,
  ],
})
export class AppModule {}
```

### Use in service

```typescript
// Before: configService.get('JWT_SECRET')
// After: configService.get('jwt.secret')

constructor(private configService: ConfigService) {}

getJwtSecret() {
  return this.configService.get('jwt.secret');
}

getAppPort() {
  return this.configService.get('app.port');
}
```

---

## Step 6: Environment-Specific `.env` Files

Create separate files for different environments:

### `.env.development`

```bash
NODE_ENV=development
APP_PORT=3000
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=password
DATABASE_NAME=nestjs_db
JWT_SECRET=dev-secret-key-change-in-production
```

### `.env.production`

```bash
NODE_ENV=production
APP_PORT=80
DATABASE_HOST=prod-db.example.com
DATABASE_PORT=5432
DATABASE_USERNAME=prod_user
DATABASE_PASSWORD=${DB_PASSWORD} # Set via environment
DATABASE_NAME=nestjs_prod
JWT_SECRET=${JWT_SECRET} # Set via environment
```

### `.env.test`

```bash
NODE_ENV=test
APP_PORT=3001
DATABASE_HOST=localhost
DATABASE_PORT=5433
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=password
DATABASE_NAME=nestjs_test
JWT_SECRET=test-secret-key
```

### `src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  const port = configService.get('app.port');

  await app.listen(port);
  console.log(`Application running on port ${port}`);
}

bootstrap();
```

---

## Step 7: Validation (Ensure Required Env Vars Exist)

### `src/env.validation.ts`

```typescript
import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, IsString, validateSync } from 'class-validator';

enum Environment {
  Development = 'development',
  Production = 'production',
  Test = 'test',
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  APP_PORT: number;

  @IsString()
  DATABASE_HOST: string;

  @IsNumber()
  DATABASE_PORT: number;

  @IsString()
  DATABASE_USERNAME: string;

  @IsString()
  DATABASE_PASSWORD: string;

  @IsString()
  DATABASE_NAME: string;

  @IsString()
  JWT_SECRET: string;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }

  return validatedConfig;
}
```

### Update `src/app.module.ts`

```typescript
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      validate, // Validate on startup
      load: [databaseConfig, jwtConfig, appConfig],
    }),
    // ...
  ],
})
export class AppModule {}
```

Now if `.env` is missing required vars, app won't start!

---

## Step 8: Test It

### Run with development config

```bash
npm run start:dev
# Uses .env
```

### Run with production config

```bash
NODE_ENV=production npm run start
# Uses .env.production
```

### Verify config is loaded

```typescript
import { Controller, Get } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Controller('config')
export class ConfigController {
  constructor(private configService: ConfigService) {}

  @Get()
  getConfig() {
    return {
      environment: this.configService.get('app.environment'),
      port: this.configService.get('app.port'),
      // Don't expose secrets!
    };
  }
}
```

---

## Security Best Practices

### 1) Never commit `.env` to git

```bash
git add .gitignore
echo ".env" >> .gitignore
git rm --cached .env
```

### 2) Create `.env.example` for reference

```bash
# .env.example (commit this to git)
DATABASE_HOST=
DATABASE_PORT=
DATABASE_USERNAME=
DATABASE_PASSWORD=
JWT_SECRET=
```

Contributors see what vars are needed without seeing actual values.

### 3) Use secrets management in production

Don't store secrets in `.env` files on servers. Use:
- **AWS Secrets Manager**
- **HashiCorp Vault**
- **Azure Key Vault**
- **GitHub Secrets** (for CI/CD)

### 4) Rotate secrets regularly

Change JWT_SECRET and DATABASE_PASSWORD periodically.

### 5) Log carefully

```typescript
// BAD - logs secrets
console.log(process.env.JWT_SECRET);

// GOOD - logs non-sensitive info
console.log(`JWT secret length: ${process.env.JWT_SECRET?.length}`);
```

---

## Common Patterns

### Optional values with defaults

```typescript
export default registerAs('app', () => ({
  port: parseInt(process.env.APP_PORT, 10) || 3000,
  debug: process.env.DEBUG === 'true',
  logLevel: process.env.LOG_LEVEL || 'info',
}));
```

### Conditional configuration

```typescript
const config = {
  synchronize: process.env.NODE_ENV === 'development',
  logging: process.env.NODE_ENV === 'development',
  dropSchemaOnInit: process.env.NODE_ENV === 'test',
};
```

---

## Troubleshooting

**"Cannot find module '.env'"**
- `.env` file doesn't exist. Create it with required variables.

**"Process.env values are undefined"**
- ConfigModule hasn't loaded yet. Use `forRootAsync` instead of `forRoot`.
- Ensure `isGlobal: true` so ConfigService is available everywhere.

**"Validation failed"**
- A required env var is missing or wrong type. Check `.env` file.

---

## Summary

You now have:
- ✅ Environment variables management
- ✅ Separate configs per environment
- ✅ Secrets not in code
- ✅ Type-safe configuration

**Next steps:**
- Learn about logging (Winston, Pino)
- Implement caching strategies
- Add rate limiting to API
- Deploy to production!

---

## Quick Reference Table

| Scenario | File | How to |
|----------|------|--------|
| Local dev | `.env` | `npm run start:dev` |
| Testing | `.env.test` | `NODE_ENV=test npm test` |
| Production | Env vars | Docker / Kubernetes env |
| Secrets | Vault/Secrets Manager | CI/CD pipeline |
| Example | `.env.example` | Commit to git |
