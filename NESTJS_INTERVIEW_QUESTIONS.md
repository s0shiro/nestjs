# NestJS Interview Questions & Answers

A comprehensive guide with common NestJS interview questions and detailed answers.

---

## Table of Contents

- [Beginner Level](#beginner-level)
- [Intermediate Level](#intermediate-level)
- [Advanced Level](#advanced-level)

---

## Beginner Level

### Q1: What is NestJS and why should we use it?

**Answer:**

NestJS is a progressive Node.js framework for building efficient, scalable, and reliable server-side applications. It's built on top of Express and uses TypeScript.

**Why use NestJS:**

1. **Type Safety** - Built on TypeScript, catching errors at compile time
2. **Architecture** - Enforces a modular, scalable architecture (similar to Angular)
3. **Dependency Injection** - Built-in DI container makes testing easier
4. **Decorators** - Makes code cleaner and more readable
5. **Best Practices** - Forces developers to follow SOLID principles
6. **Full-Stack Framework** - Includes everything needed for enterprise applications
7. **Large Ecosystem** - Rich library support and community

**Example:**
```typescript
// Simple NestJS application
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
  console.log(`Application running on: http://localhost:3000`);
}
bootstrap();
```

---

### Q2: What is the difference between a Module, Controller, and Service?

**Answer:**

| Component | Purpose | Responsibility |
|-----------|---------|-----------------|
| **Module** | Organizing related features | Encapsulate controllers, services, and other features |
| **Controller** | Handle HTTP requests | Receive requests, delegate logic, return responses |
| **Service** | Business logic container | Contain reusable business logic and data access |

**Example:**

```typescript
// Service - Contains business logic
@Injectable()
export class UsersService {
  private users = [];

  create(createUserDto: CreateUserDto) {
    const user = { id: 1, ...createUserDto };
    this.users.push(user);
    return user;
  }

  findAll() {
    return this.users;
  }
}

// Controller - Handles HTTP requests
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}

// Module - Organizes everything
@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // Export service for use in other modules
})
export class UsersModule {}
```

---

### Q3: What are Decorators and why are they important in NestJS?

**Answer:**

Decorators are functions that add metadata and functionality to classes, methods, properties, and parameters. They're essential in NestJS for declaring and configuring components.

**Why Important:**

1. Declarative programming style
2. Reduces boilerplate code
3. Adds metadata for framework processing
4. Makes code more readable and maintainable

**Common Decorators:**

```typescript
// Class Decorators
@Controller('posts')
@Injectable()
@Module()

// Method Decorators
@Get()
@Post()
@Put()
@Delete()
@UseGuards(AuthGuard)
@UseInterceptors(LoggingInterceptor)

// Parameter Decorators
@Param('id')
@Query('limit')
@Body()
@Headers('authorization')
@Req()
@Res()
```

**Example:**

```typescript
@Controller('posts')
export class PostsController {
  constructor(private postsService: PostsService) {}

  // Method decorator with parameter decorators
  @Get(':id')
  getPost(@Param('id') id: string) {
    return this.postsService.findOne(id);
  }

  @Post()
  createPost(@Body() createPostDto: CreatePostDto) {
    return this.postsService.create(createPostDto);
  }
}
```

---

### Q4: What is Dependency Injection and how does it work in NestJS?

**Answer:**

Dependency Injection (DI) is a design pattern where a class receives its dependencies from external sources rather than creating them itself. This promotes loose coupling and testability.

**How it works in NestJS:**

```typescript
// Service 1
@Injectable()
export class DatabaseService {
  connect() {
    console.log('Connected to database');
  }
}

// Service 2
@Injectable()
export class UsersService {
  // DatabaseService is injected through constructor
  constructor(private dbService: DatabaseService) {}

  createUser(user: User) {
    this.dbService.connect();
    // Create user...
  }
}

// Controller
@Controller('users')
export class UsersController {
  // Both services are automatically injected
  constructor(private usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.createUser(createUserDto);
  }
}

// Module - Register providers
@Module({
  controllers: [UsersController],
  providers: [UsersService, DatabaseService],
})
export class UsersModule {}
```

**Benefits:**

1. **Testability** - Easy to inject mocks
2. **Loose Coupling** - Classes don't depend on concrete implementations
3. **Maintainability** - Easy to swap implementations
4. **Reusability** - Services can be used across modules

---

### Q5: How do you handle errors in NestJS?

**Answer:**

NestJS provides built-in HTTP exceptions and a global exception filter mechanism for centralized error handling.

**Using HTTP Exceptions:**

```typescript
@Get(':id')
getUser(@Param('id') id: string) {
  const user = this.usersService.findOne(id);
  
  if (!user) {
    throw new NotFoundException('User not found');
  }
  
  return user;
}
```

**Common HTTP Exceptions:**

```typescript
throw new BadRequestException('Invalid data');        // 400
throw new UnauthorizedException('Not authorized');   // 401
throw new ForbiddenException('Access denied');        // 403
throw new NotFoundException('Resource not found');    // 404
throw new ConflictException('Conflict occurred');     // 409
throw new InternalServerErrorException('Error');      // 500
```

**Custom Exception Filter:**

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      message: exception.getResponse(),
      timestamp: new Date().toISOString(),
    });
  }
}

// Use globally
app.useGlobalFilters(new HttpExceptionFilter());
```

---

## Intermediate Level

### Q6: What are Pipes and how do you create custom pipes?

**Answer:**

Pipes transform and validate data before it reaches the route handler. They implement the `PipeTransform` interface.

**Built-in Pipes:**

```typescript
@Get(':id')
getUser(
  @Param('id', ParseIntPipe) id: number,
  @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number
) {
  // id is guaranteed to be a number
  // limit defaults to 10 if not provided
  return this.usersService.findUsers(id, limit);
}
```

**Creating Custom Pipes:**

```typescript
@Injectable()
export class ValidateUserPipe implements PipeTransform {
  transform(value: CreateUserDto, metadata: ArgumentMetadata): CreateUserDto {
    if (!value.email || !value.name) {
      throw new BadRequestException('Email and name are required');
    }
    
    // Validate email format
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value.email)) {
      throw new BadRequestException('Invalid email format');
    }

    return value;
  }
}

// Use in controller
@Post()
createUser(@Body(ValidateUserPipe) createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

**Using class-validator:**

```typescript
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(3)
  name: string;

  @IsOptional()
  @IsPhoneNumber()
  phone?: string;
}

@Post()
createUser(
  @Body(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
  createUserDto: CreateUserDto
) {
  return this.usersService.create(createUserDto);
}
```

---

### Q7: What are Guards and how do you implement authentication?

**Answer:**

Guards determine whether a request should be handled based on certain conditions (authorization, authentication, etc.). They implement the `CanActivate` interface.

**Creating an Authentication Guard:**

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      const payload = jwt.verify(token, 'your-secret-key');
      request.user = payload;
      return true;
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// Use on controller
@Controller('users')
@UseGuards(AuthGuard)
export class UsersController {
  @Get()
  getUsers(@Req() request: Request) {
    console.log(request.user); // User from guard
    return this.usersService.findAll();
  }
}
```

**Role-Based Guard:**

```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const requiredRoles = Reflect.getMetadata('roles', context.getHandler());

    if (!requiredRoles) {
      return true; // No role requirement
    }

    return requiredRoles.includes(user.role);
  }
}

// Create decorator for roles
export const Roles = (...roles: string[]) =>
  SetMetadata('roles', roles);

// Use in controller
@Get('admin')
@Roles('admin')
@UseGuards(AuthGuard, RolesGuard)
getAdminData() {
  return { message: 'Admin data' };
}
```

---

### Q8: What are Interceptors and when do you use them?

**Answer:**

Interceptors intercept the request-response cycle and can add functionality before and after route handlers execute. They implement `NestInterceptor`.

**Use Cases:**

- Logging requests/responses
- Caching responses
- Transforming responses
- Error handling
- Performance monitoring

**Example - Logging Interceptor:**

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const now = Date.now();

    console.log(`${method} ${url}`);

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - now;
        console.log(`${method} ${url} - ${duration}ms`);
      }),
      catchError((error) => {
        console.error(`Error: ${method} ${url} - ${error.message}`);
        throw error;
      }),
    );
  }
}

// Apply globally
app.useGlobalInterceptors(new LoggingInterceptor());

// Or to a controller
@Controller('users')
@UseInterceptors(LoggingInterceptor)
export class UsersController {}
```

**Example - Caching Interceptor:**

```typescript
@Injectable()
export class CachingInterceptor implements NestInterceptor {
  private cache = new Map();

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = `${request.method}-${request.url}`;

    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey));
    }

    return next.handle().pipe(
      tap((data) => {
        this.cache.set(cacheKey, data);
      }),
    );
  }
}
```

---

### Q9: How do you test a NestJS application?

**Answer:**

NestJS provides testing utilities with Jest (default test framework).

**Unit Test Example:**

```typescript
describe('UsersService', () => {
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  describe('create', () => {
    it('should create a user', () => {
      const createUserDto: CreateUserDto = {
        email: 'test@example.com',
        name: 'Test User',
      };

      const result = service.create(createUserDto);
      expect(result).toEqual(expect.objectContaining(createUserDto));
    });
  });

  describe('findAll', () => {
    it('should return an array of users', () => {
      const result = service.findAll();
      expect(Array.isArray(result)).toBe(true);
    });
  });
});
```

**Controller Test Example:**

```typescript
describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: {
            create: jest.fn(),
            findAll: jest.fn().mockReturnValue([
              { id: 1, email: 'test@example.com', name: 'Test' },
            ]),
          },
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  describe('findAll', () => {
    it('should return an array of users', async () => {
      const result = await controller.findAll();
      expect(result).toEqual([
        { id: 1, email: 'test@example.com', name: 'Test' },
      ]);
      expect(service.findAll).toHaveBeenCalled();
    });
  });
});
```

---

### Q10: What is the purpose of DTOs (Data Transfer Objects)?

**Answer:**

DTOs are classes that define the shape of data being transferred between processes. They're used for:

1. **Validation** - Ensure incoming data matches expectations
2. **Documentation** - Define API contract
3. **Type Safety** - TypeScript type checking
4. **Security** - Whitelist fields to prevent overposting

**Example:**

```typescript
// Create DTO
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(3)
  @MaxLength(50)
  name: string;

  @IsOptional()
  @IsPhoneNumber()
  phone?: string;

  @IsOptional()
  @IsString()
  bio?: string;
}

// Update DTO - often extends CreateUserDto
export class UpdateUserDto extends PartialType(CreateUserDto) {}

// Response DTO - what we return to client
export class UserResponseDto {
  id: number;
  email: string;
  name: string;
  createdAt: Date;
  // Don't include sensitive fields like password
}

// Usage in controller
@Controller('users')
export class UsersController {
  @Post()
  create(@Body() createUserDto: CreateUserDto): UserResponseDto {
    return this.usersService.create(createUserDto);
  }

  @Put(':id')
  update(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto
  ): UserResponseDto {
    return this.usersService.update(id, updateUserDto);
  }
}
```

---

## Advanced Level

### Q11: How do you implement middleware in NestJS?

**Answer:**

Middleware is a function or class that runs before route handlers. Used for logging, authentication, request transformation, etc.

**Functional Middleware:**

```typescript
export function loggingMiddleware(req: Request, res: Response, next: NextFunction) {
  console.log(`${req.method} ${req.path}`);
  next();
}

// Apply in module
@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(loggingMiddleware)
      .forRoutes('*'); // Apply to all routes
  }
}
```

**Class-based Middleware:**

```typescript
@Injectable()
export class AuthMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      throw new UnauthorizedException('No token');
    }

    try {
      req.user = jwt.verify(token, 'secret');
      next();
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// Apply selectively
configure(consumer: MiddlewareConsumer) {
  consumer
    .apply(AuthMiddleware)
    .forRoutes(
      { path: 'users', method: RequestMethod.GET },
      { path: 'users/:id', method: RequestMethod.PUT }
    )
    .apply(loggingMiddleware)
    .forRoutes('*');
}
```

---

### Q12: What is the difference between @Injectable and @Component decorators?

**Answer:**

`@Injectable()` is used to mark a class as a provider that can be injected into other classes (services, guards, pipes, interceptors, etc.).

`@Component` doesn't exist in NestJS (that's Angular). In NestJS:

- **@Injectable()** - For any injectable class (services, guards, pipes, filters)
- **@Controller()** - For classes that handle HTTP requests
- **@Module()** - For module classes

**Example:**

```typescript
// Service - uses @Injectable
@Injectable()
export class UsersService {
  findAll() {
    return [];
  }
}

// Guard - uses @Injectable
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    return true;
  }
}

// Pipe - uses @Injectable
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any): any {
    return value;
  }
}

// Interceptor - uses @Injectable
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    return next.handle();
  }
}
```

---

### Q13: How do you handle database connections in NestJS?

**Answer:**

NestJS is database-agnostic. You can use TypeORM, Prisma, Mongoose, etc.

**Example with TypeORM:**

```typescript
// Install: npm install @nestjs/typeorm typeorm mysql2

// database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'password',
      database: 'nestjs_db',
      entities: [__dirname + '/../**/*.entity{.ts,.js}'],
      synchronize: true,
    }),
  ],
})
export class DatabaseModule {}
```

**Entity Definition:**

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'varchar', length: 100 })
  name: string;

  @Column({ type: 'varchar', unique: true })
  email: string;

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;
}
```

**Service with Database Access:**

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  create(createUserDto: CreateUserDto): Promise<User> {
    const user = this.usersRepository.create(createUserDto);
    return this.usersRepository.save(user);
  }

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User> {
    return this.usersRepository.findOne({ where: { id } });
  }

  async update(id: number, updateUserDto: UpdateUserDto): Promise<User> {
    await this.usersRepository.update(id, updateUserDto);
    return this.findOne(id);
  }

  remove(id: number): Promise<void> {
    return this.usersRepository.delete(id).then(() => undefined);
  }
}
```

**Module Setup:**

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

---

### Q14: How do you implement validation at the global level?

**Answer:**

Use the global ValidationPipe in your main.ts file to validate all incoming requests.

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,                    // Remove unknown properties
      forbidNonWhitelisted: true,         // Throw error if unknown properties
      skipMissingProperties: false,       // Validate missing properties
      transform: true,                    // Automatically transform payloads
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  // Global exception filter
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(3000);
}
bootstrap();
```

**Benefits:**

- Centralized validation
- Consistent error responses
- Automatic type transformation
- Security (whitelist fields)

---

### Q15: How do you handle WebSocket connections in NestJS?

**Answer:**

NestJS provides `@nestjs/websockets` for real-time communication.

```typescript
// Install: npm install @nestjs/websockets @nestjs/platform-socket.io socket.io

@WebSocketGateway()
export class ChatGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  afterInit(server: Server) {
    console.log('WebSocket server initialized');
  }

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
    this.server.emit('userJoined', { message: 'A user joined' });
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  @SubscribeMessage('message')
  handleMessage(client: Socket, data: any): void {
    this.server.emit('message', data);
  }

  @SubscribeMessage('notification')
  handleNotification(client: Socket, payload: any): WsResponse<any> {
    return { event: 'notification', data: payload };
  }
}

// Module
@Module({
  providers: [ChatGateway],
})
export class ChatModule {}
```

**Client-side (Socket.io):**

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000');

socket.on('connect', () => {
  console.log('Connected to server');
});

socket.emit('message', { text: 'Hello from client' });

socket.on('message', (data) => {
  console.log('Received:', data);
});
```

---

## Bonus Questions

### Q16: What are the differences between `@Req()` and `@Request()`?

**Answer:**

They're aliases for the same thing. Both give you access to the Express Request object.

```typescript
@Get()
getUser(@Req() request: Request) {
  // Both work the same
  console.log(request.method);
  console.log(request.url);
}

// Same as:
@Get()
getUser(@Request() request: Request) {
  console.log(request.method);
}
```

---

### Q17: How do you configure CORS in NestJS?

**Answer:**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableCors({
    origin: 'http://localhost:3000',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
  });

  await app.listen(3000);
}
bootstrap();
```

---

### Q18: What is a Singleton pattern and how is it used in NestJS?

**Answer:**

A Singleton ensures only one instance of a class exists. NestJS providers are Singletons by default.

```typescript
@Injectable() // Creates single instance
export class ConfigService {
  private config = {};

  getConfig() {
    return this.config;
  }
}

// Only one instance is created and shared across the app
@Controller('config')
export class ConfigController {
  constructor(
    private config1: ConfigService,
    private config2: ConfigService  // Same instance
  ) {}

  @Get()
  getConfig() {
    // config1 and config2 are the same object
    console.log(this.config1 === this.config2); // true
  }
}
```

---

### Q19: How do you implement request/response logging?

**Answer:**

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, body } = request;

    this.logger.log(`Incoming Request: ${method} ${url}`);
    this.logger.debug(`Request Body:`, body);

    const now = Date.now();

    return next.handle().pipe(
      tap((data) => {
        const duration = Date.now() - now;
        this.logger.log(`Response sent in ${duration}ms`);
        this.logger.debug(`Response Data:`, data);
      }),
      catchError((error) => {
        const duration = Date.now() - now;
        this.logger.error(`Error in ${duration}ms:`, error.message);
        throw error;
      }),
    );
  }
}
```

---

### Q20: What is the purpose of `@nestjs/config`?

**Answer:**

Manages environment variables and configuration in a centralized way.

```typescript
// Install: npm install @nestjs/config

// .env
DATABASE_HOST=localhost
DATABASE_PORT=5432
API_KEY=secret123

// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
  ],
})
export class AppModule {}

// Use in service
@Injectable()
export class DatabaseService {
  constructor(private configService: ConfigService) {}

  getConnection() {
    const host = this.configService.get('DATABASE_HOST');
    const port = this.configService.get('DATABASE_PORT');
    
    return `${host}:${port}`;
  }
}
```

---

## Summary Tips for NestJS Interviews

✅ **Understand the architecture** - Modules, Controllers, Services, Guards, Pipes, Interceptors, Filters

✅ **Dependency Injection** - Know why it's important and how to use it

✅ **Testing** - Be able to write unit tests with mocks

✅ **Error Handling** - Know how to use HTTP exceptions and custom filters

✅ **Decorators** - Understand what they do and how to use them

✅ **Real-world scenarios** - Be prepared to solve problems like database integration, authentication, WebSocket

✅ **Code Quality** - Show knowledge of DTOs, validation, and best practices

✅ **Performance** - Understand caching, interceptors, and middleware

---

Good luck with your NestJS interview! 🚀
