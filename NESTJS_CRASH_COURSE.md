# NestJS Crash Course - Learn in 25 Minutes

A practical guide to building your first NestJS application by creating a Podcast API.

---

## Table of Contents

1. [Why NestJS?](#why-nestjs)
2. [Setting Up](#setting-up)
3. [Core Concepts](#core-concepts)
4. [Modules](#modules)
5. [Decorators](#decorators)
6. [Controllers](#controllers)
7. [Providers & Services](#providers--services)
8. [Dependency Injection](#dependency-injection)
9. [Testing](#testing)
10. [Error Handling](#error-handling)
11. [Pipes](#pipes)
12. [Guards](#guards)

---

## Why NestJS?

NestJS achieves something remarkable: **it makes it near impossible to write bad code**. Here's why:

- **Rigid Architecture**: Enforces best practices through its framework structure
- **Scalable**: Build large applications with confidence
- **Testable**: Dependency injection makes unit testing straightforward
- **Loosely Coupled**: Components are independent and reusable
- **Type-Safe**: Built on TypeScript

The framework forces you to follow patterns that result in well-organized, maintainable code.

---

## Setting Up

### Installation

Install the NestJS CLI globally:
```bash
npm install -g @nestjs/cli
```

### Create a New Project

```bash
nest new podcast-app
cd podcast-app
```

### Start Development Server

```bash
npm run start:dev
```

Visit `http://localhost:3000` and you should see "Hello World!"

---

## Core Concepts

Every NestJS application has **10 key concepts** that work together:

1. **Modules** - Organize your application
2. **Decorators** - Enhance classes and methods
3. **Controllers** - Handle HTTP requests
4. **Providers** - Contain business logic
5. **Dependency Injection** - Wire dependencies
6. **Testing** - Built-in test utilities
7. **Error Handling** - Manage exceptions
8. **Pipes** - Validate and transform data
9. **Guards** - Control access
10. **Interceptors** - Hook into lifecycle (covered later)

---

## Modules

### What Are Modules?

Modules are the **building blocks** of your NestJS application. Every app has:
- One **root module** that boots the application
- Feature modules that encapsulate functionality
- Modules can import other modules

### Architecture Example

```
Root App Module
├── Episodes Module
├── Topics Module
└── Config Module
```

### Creating Modules

Use the CLI to generate modules:
```bash
nest generate module episodes
nest generate module topics
nest generate module config
```

### Module Structure

The generated module file:
```typescript
@Module({
  imports: [],      // Other modules this module uses
  controllers: [],  // Route handlers
  providers: [],    // Services and utilities
  exports: []       // What this module shares with others
})
export class EpisodesModule {}
```

### Main Application File

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

---

## Decorators

### Understanding Decorators

Decorators are **not magic** — they're just functions that enhance classes, methods, and parameters.

Think of decorators as **adding accessories or outfits** that give objects extra power:
- A naked class + `@Module()` decorator = can be used by other modules
- Functions get **extra functionality** and **metadata**

### Why Decorators Matter

In NestJS, decorators are everywhere. Understanding them is critical to using the framework.

### Common Decorators

| Decorator | Use |
|-----------|-----|
| `@Module()` | Marks a class as a module |
| `@Controller()` | Marks a class as a controller |
| `@Injectable()` | Marks a class as a provider |
| `@Get()`, `@Post()`, etc. | HTTP method handlers |
| `@Param()`, `@Query()`, `@Body()` | Extract request data |
| `@UseGuards()`, `@UseInterceptors()` | Apply middleware |

---

## Controllers

### What Are Controllers?

Controllers are **entry points** for incoming requests. Their job:
1. Receive HTTP requests
2. Extract data from the request
3. Call services to do work
4. Return responses

### Structure

```typescript
@Controller('episodes')
export class EpisodesController {
  constructor(private readonly episodesService: EpisodesService) {}

  @Get()
  findAll() {
    return this.episodesService.findAll();
  }

  @Get('featured')
  findFeatured() {
    return this.episodesService.findFeatured();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.episodesService.findOne(id);
  }

  @Post()
  create(@Body() createEpisodeDto: CreateEpisodeDto) {
    return this.episodesService.create(createEpisodeDto);
  }
}
```

### Key Points

- `@Controller('episodes')` defines the root path
- `@Get()`, `@Post()`, etc. define HTTP methods
- Parameters can be extracted with `@Param()`, `@Query()`, `@Body()`
- Controllers **delegate work** to services

### Routes Example

With the above controller:
- `GET /episodes` → Get all episodes
- `GET /episodes/featured` → Get featured episodes
- `GET /episodes/:id` → Get episode by ID
- `POST /episodes` → Create new episode

---

## Providers & Services

### What Are Providers?

Providers are classes with the `@Injectable()` decorator. They contain:
- Business logic
- Database access
- Utility functions
- Any reusable logic

### Creating a Service

```bash
nest generate service episodes
```

### Service Implementation

```typescript
@Injectable()
export class EpisodesService {
  private episodes = [
    { id: 1, title: 'Episode 1', featured: true },
    { id: 2, title: 'Episode 2', featured: false },
  ];

  findAll() {
    return this.episodes;
  }

  findFeatured() {
    return this.episodes.filter(ep => ep.featured);
  }

  findOne(id: string) {
    return this.episodes.find(ep => ep.id === Number(id));
  }

  create(createEpisodeDto: CreateEpisodeDto) {
    const newEpisode = {
      id: this.episodes.length + 1,
      ...createEpisodeDto,
    };
    this.episodes.push(newEpisode);
    return newEpisode;
  }
}
```

### Registering Providers

The CLI automatically adds services to the module's `providers` array:

```typescript
@Module({
  controllers: [EpisodesController],
  providers: [EpisodesService], // ← Added here
})
export class EpisodesModule {}
```

**Important**: If you create providers manually, always add them to the `providers` array!

---

## Dependency Injection

### What Is It?

Dependency Injection is a design pattern where:
- A class doesn't create its own dependencies
- Dependencies are **injected** by the framework
- Makes code **testable** and **loosely coupled**

### How It Works

```typescript
@Controller('episodes')
export class EpisodesController {
  // NestJS will create and inject EpisodesService
  constructor(private readonly episodesService: EpisodesService) {}
}
```

When you declare a parameter of a specific type in the constructor, NestJS:
1. Creates an instance of that type
2. Injects it into your class
3. Stores it as a property (because of `private`)

### Using Injected Services

```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  // Use the injected service
  return this.episodesService.findOne(id);
}
```

### Injecting Services From Other Modules

To use a service from another module:

1. **Export the service** from the config module:
```typescript
@Module({
  providers: [ConfigService],
  exports: [ConfigService],  // ← Export it
})
export class ConfigModule {}
```

2. **Import the module** where you need it:
```typescript
@Module({
  imports: [ConfigModule], // ← Import it
  controllers: [EpisodesController],
  providers: [EpisodesService],
})
export class EpisodesModule {}
```

3. **Inject the service**:
```typescript
@Injectable()
export class EpisodesService {
  constructor(private readonly configService: ConfigService) {}
}
```

---

## Testing

### Why Testing Matters

Dependency injection makes testing **easy** because you can inject mock objects instead of real services.

### Generated Test File

```typescript
describe('EpisodesController', () => {
  let controller: EpisodesController;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [EpisodesController],
      providers: [EpisodesService],
    }).compile();

    controller = module.get<EpisodesController>(EpisodesController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });
});
```

### Testing With Mocks

```typescript
describe('EpisodesController', () => {
  let controller: EpisodesController;
  let service: EpisodesService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [EpisodesController],
      providers: [
        {
          provide: EpisodesService,
          useValue: {
            findOne: jest.fn().mockReturnValue({ id: 1, title: 'Episode 1' }),
          },
        },
      ],
    }).compile();

    controller = module.get<EpisodesController>(EpisodesController);
    service = module.get<EpisodesService>(EpisodesService);
  });

  describe('findOne', () => {
    it('should return an episode', () => {
      const result = controller.findOne('1');
      expect(result).toEqual({ id: 1, title: 'Episode 1' });
    });

    it('should call service with correct id', () => {
      controller.findOne('1');
      expect(service.findOne).toHaveBeenCalledWith('1');
    });
  });

  describe('when episode is not found', () => {
    it('should throw NotFoundException', () => {
      jest.spyOn(service, 'findOne').mockReturnValue(null);
      expect(() => controller.findOne('999')).toThrow(NotFoundException);
    });
  });
});
```

---

## Error Handling

### Default Behavior

If an unhandled error occurs, NestJS returns:
```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

### Using HTTP Exceptions

Throw meaningful exceptions:

```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  const episode = this.episodesService.findOne(id);
  
  if (!episode) {
    throw new NotFoundException('Episode not found');
  }
  
  return episode;
}
```

### Built-in HTTP Exceptions

```typescript
throw new NotFoundException('Resource not found');     // 404
throw new UnauthorizedException('Not authorized');    // 401
throw new ForbiddenException('Access forbidden');      // 403
throw new BadRequestException('Invalid input');        // 400
throw new InternalServerErrorException('Server error');// 500
throw new ConflictException('Resource already exists');// 409
```

### Custom Exception Response

```typescript
throw new HttpException(
  {
    statusCode: 400,
    message: 'Invalid episode data',
    errors: ['Title is required', 'Description is too short']
  },
  HttpStatus.BAD_REQUEST
);
```

---

## Pipes

### What Are Pipes?

Pipes validate and/or transform incoming request data before it reaches the handler.

Two main responsibilities:
1. **Validation** - Ensure data meets requirements
2. **Transformation** - Convert data to expected type

### Built-in Pipes

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number  // Convert string to number
) {
  return this.episodesService.findOne(id);
}
```

### Pipe Options

```typescript
@Get(':limit')
findAll(
  @Query('limit', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  limit: number
) {
  return this.episodesService.findAll(limit);
}
```

### Using Default Values

```typescript
@Get()
findAll(
  @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number
) {
  return this.episodesService.findAll(limit);
}
```

### Creating Custom Pipes

```typescript
@Injectable()
export class IsStrictlyPositivePipe implements PipeTransform {
  transform(value: number, metadata: ArgumentMetadata): number {
    if (value < 0) {
      throw new BadRequestException('Value must be positive');
    }
    return value;
  }
}
```

Use it:
```typescript
@Get()
findAll(@Query('limit', IsStrictlyPositivePipe) limit: number) {
  return this.episodesService.findAll(limit);
}
```

### Validating Request Bodies

#### Install Dependencies

```bash
npm install class-validator class-transformer
```

#### Create a DTO (Data Transfer Object)

```typescript
import { IsString, IsBoolean, IsOptional, IsDate } from 'class-validator';
import { Type } from 'class-transformer';

export class CreateEpisodeDto {
  @IsString()
  title: string;

  @IsBoolean()
  @IsOptional()
  featured?: boolean;

  @Type(() => Date)
  @IsDate()
  @IsOptional()
  publishedAt?: Date;
}
```

#### Add ValidationPipe to Controller

```typescript
@Post()
create(
  @Body(ValidationPipe) createEpisodeDto: CreateEpisodeDto
) {
  return this.episodesService.create(createEpisodeDto);
}
```

Now if the request body doesn't match the DTO, it will automatically throw a validation error!

---

## Guards

### What Are Guards?

Guards determine **whether a request should be accepted or rejected** based on specific conditions:
- Authentication status
- User roles
- Permissions
- Custom logic

### Creating a Guard

```typescript
@Injectable()
export class ApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];
    
    return apiKey === 'your-secret-api-key';
  }
}
```

### Applying Guards

#### Guard Entire Controller

```typescript
@Controller('episodes')
@UseGuards(ApiKeyGuard)
export class EpisodesController {
  // All routes require API key
}
```

#### Guard Specific Route

```typescript
@Controller('episodes')
export class EpisodesController {
  @Post()
  @UseGuards(ApiKeyGuard)
  create(@Body() createEpisodeDto: CreateEpisodeDto) {
    return this.episodesService.create(createEpisodeDto);
  }

  @Get()
  findAll() {
    // This route doesn't require API key
    return this.episodesService.findAll();
  }
}
```

### Guard Behavior

- **Returns `true`** → Request proceeds to handler
- **Returns `false`** → Request is rejected with 403 Forbidden
- **Throws Exception** → Request is rejected with that exception

---

## Request Lifecycle

Here's how a request flows through NestJS:

```
1. Request arrives
   ↓
2. Middleware processes (if any)
   ↓
3. Guards check authorization
   ↓
4. Interceptors pre-processing (if any)
   ↓
5. Pipes validate & transform data
   ↓
6. Route handler executes
   ↓
7. Interceptors post-processing (if any)
   ↓
8. Exception filters catch errors (if any)
   ↓
9. Response sent to client
```

---

## Next Steps

Now that you understand the crash course concepts:

1. **Build the Podcast API** - Implement all the endpoints with proper validation and error handling
2. **Add Authentication** - Learn how to secure your API with JWT tokens
3. **Database Integration** - Connect to a real database (PostgreSQL, MongoDB, etc.)
4. **Advanced Features** - Learn about interceptors, custom decorators, and middleware
5. **Deployment** - Deploy your NestJS app to production

---

## Key Takeaways

✅ NestJS enforces architecture that makes good code inevitable

✅ Modules organize your app into feature groups

✅ Decorators enhance classes and methods with extra power

✅ Controllers handle requests, services contain logic

✅ Dependency injection wires everything together and enables testing

✅ Pipes validate and transform data before handlers

✅ Guards control who can access what

✅ Error handling is built-in and customizable

---

## Common CLI Commands

```bash
# Generate new module
nest generate module feature-name

# Generate controller
nest generate controller feature-name

# Generate service
nest generate service feature-name

# Generate guard
nest generate guard feature-name

# Generate pipe
nest generate pipe feature-name

# Shortcuts
nest g mo feature-name     # Generate module
nest g co feature-name     # Generate controller
nest g s feature-name      # Generate service
nest g gu feature-name     # Generate guard
nest g pi feature-name     # Generate pipe
```

---

## Resources

- NestJS Official Documentation: https://docs.nestjs.com
- NestJS Course: https://go.tech-vision.io/nestjs-course-yt-vid-17
- Class Validator: https://github.com/typestack/class-validator
- Class Transformer: https://github.com/typestack/class-transformer

Happy coding! 🚀
