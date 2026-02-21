# Testing Patterns

**Analysis Date:** 2026-02-21

This codebase contains comprehensive testing for the backend API. The frontend does not have tests configured.

## Backend (API) Testing

### Test Framework

**Runner:**
- Jest (v30.2.0)
- Config: Package.json `jest` field and `jest.config.*` patterns

**Assertion Library:**
- Jest built-in assertions (`expect()`)

**Test Commands:**
```bash
npm run test              # Run all tests with coverage
npm run test:watch       # Watch mode for development
npm run test:debug       # Debug mode with inspector
```

### Test File Organization

**Location:** Co-located in `test/` subdirectories within each module

**Structure:**
```
src/[feature]/
├── services/
│   └── [name].service.ts
├── controllers/
│   └── [name].controller.ts
├── repositories/
│   └── [name].repository.ts
└── test/
    ├── services/
    │   └── [name].service.spec.ts
    ├── repositories/
    │   └── [name].repository.spec.ts
    ├── controllers/
    │   └── [name].controller.spec.ts
    └── mocks/
        └── [feature]-mock-generator.ts
```

**Naming Pattern:**
- Test files: `[name].spec.ts` → `business-creator.service.spec.ts`, `money.value-object.spec.ts`

**Jest Configuration** from `package.json`:
```json
{
  "jest": {
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "src",
    "testRegex": ".*\\.spec\\.ts$",
    "transform": {
      "^.+\\.(t|j)s$": "ts-jest"
    },
    "collectCoverageFrom": ["**/*.(t|j)s"],
    "coverageDirectory": "../coverage",
    "testEnvironment": "node",
    "moduleNameMapper": {
      "^@auth/(.*)$": "<rootDir>/auth/$1",
      "^@business/(.*)$": "<rootDir>/business/$1",
      "^@core/(.*)$": "<rootDir>/core/$1",
      "^@customer/(.*)$": "<rootDir>/customer/$1",
      "^@email/(.*)$": "<rootDir>/email/$1",
      "^@item/(.*)$": "<rootDir>/item/$1",
      "^@job/(.*)$": "<rootDir>/job/$1",
      "^@quote/(.*)$": "<rootDir>/quote/$1",
      "^@user/(.*)$": "<rootDir>/user/$1"
    }
  }
}
```

### Test Structure

**Suite Organization Pattern:**

From `src/business/test/services/business-creator.service.spec.ts`:

```typescript
describe("BusinessCreator", () => {
  let businessCreator: BusinessCreator;
  let businessRepository: jest.Mocked<BusinessRepository>;
  let authorizedCreatorFactory: jest.Mocked<AuthorizedCreatorFactory>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BusinessCreator,
        {
          provide: BusinessRepository,
          useValue: {
            create: jest.fn(),
            findByIdOrFail: jest.fn(),
          },
        },
        // ... other dependencies
      ],
    }).compile();

    businessCreator = module.get<BusinessCreator>(BusinessCreator);
    businessRepository = module.get(BusinessRepository);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe("create", () => {
    it("should create business successfully", async () => {
      // Arrange
      const businessData = BusinessMockGenerator.createBusinessDto();
      const user = BusinessMockGenerator.createUserDto();

      // Act
      const result = await businessCreator.create(user, businessData);

      // Assert
      expect(result).toEqual(
        expect.objectContaining({
          id: expect.any(String),
          name: businessData.name,
        }),
      );
    });

    it("should throw error when access control fails", async () => {
      // Arrange
      const accessError = new Error("Access denied");
      authorizedCreatorFactory.createFor.mockReturnValue(
        mockAuthorizedCreator,
      );

      // Act & Assert
      await expect(businessCreator.create(user, businessData)).rejects.toThrow(
        accessError,
      );
    });
  });
});
```

**Key Patterns:**
1. **Arrange-Act-Assert (AAA):** Clear three-section structure in each test
2. **Nested describe blocks:** Group related tests under method/feature names
3. **Descriptive test names:** `should [expected behavior] when [condition]`
4. **Setup/Teardown:** Use `beforeEach()` for test module initialization and `afterEach()` for cleanup
5. **Jest Mocking:** `jest.Mocked<T>` for type-safe mocks

### Mocking

**Framework:** Jest built-in mocking (`jest.fn()`, `jest.mock()`)

**Mock Generators Pattern:**

From `src/business/test/mocks/business-mock-generator.ts`:

```typescript
import { ObjectId } from "mongodb";
import { IBusinessDto } from "@business/data-transfer-objects/business.dto";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

export class BusinessMockGenerator {
  static createBusinessDto(overrides?: Partial<IBusinessDto>): IBusinessDto {
    return {
      id: new ObjectId().toString(),
      name: "Test Business",
      country: "US",
      currency: "USD",
      status: BusinessStatus.ACTIVE,
      ...overrides,
    };
  }

  static createUserDto(overrides?: Partial<IUserDto>): IUserDto {
    return {
      id: new ObjectId().toString(),
      externalAuthUserId: "firebase-uid-123",
      email: "test@example.com",
      nickname: "testuser",
      name: "Test User",
      businessIds: [],
      supportRoles: DtoCollection.empty(),
      businessRoles: DtoCollection.empty(),
      ...overrides,
    };
  }
}
```

**Mock Setup in Tests:**

```typescript
beforeEach(async () => {
  const module: TestingModule = await Test.createTestingModule({
    providers: [
      BusinessCreator,
      {
        provide: BusinessRepository,
        useValue: {
          create: jest.fn(),
          findByIdOrFail: jest.fn(),
        },
      },
      {
        provide: BusinessPolicy,
        useValue: {
          canCreate: jest.fn().mockReturnValue(true),
        },
      },
    ],
  }).compile();

  businessCreator = module.get<BusinessCreator>(BusinessCreator);
  businessRepository = module.get(BusinessRepository);
  businessPolicy = module.get(BusinessPolicy);
});
```

**What to Mock:**
- Repository methods → database operations
- Policy methods → authorization checks
- External services → SendGrid, Firebase, third-party APIs
- Factories → dependency creation

**What NOT to Mock:**
- Value objects → Money, Currency (test business logic directly)
- DTOs and data structures
- Core utilities and helpers
- Validation logic

### Value Object Testing

**Pattern from `src/core/test/value-objects/money.value-object.spec.ts`:**

```typescript
describe("Money", () => {
  const USD: Currency = "USD";

  const createMoney = (amount: number, precision?: number): Money =>
    Money.fromMajorUnits(amount, USD, precision);

  describe("fromMajorUnits", () => {
    it("stores major units and keeps conversions consistent", () => {
      const money = Money.fromMajorUnits(123.45, USD);

      expect(money.toMajorUnits()).toBeCloseTo(123.45);
      expect(money.toMinorUnits()).toBe(12345);
    });

    it("supports custom precision when translating to minor units", () => {
      const money = Money.fromMajorUnits(1.234, USD, 3);

      expect(money.toMajorUnits()).toBeCloseTo(1.234);
      expect(money.toMinorUnits()).toBe(1234);
    });
  });

  describe("add", () => {
    it("sums two money values with the same currency", () => {
      const augend = createMoney(10.25);
      const addend = createMoney(5.75);

      const result = augend.add(addend);

      expect(result.toMajorUnits()).toBeCloseTo(16);
      expect(result.toMinorUnits()).toBe(1600);
    });
  });

  describe("multiply", () => {
    it("multiplies the amount by the given factor", () => {
      const money = createMoney(12.34);

      const result = money.multiply(1.5);

      expect(result.toMajorUnits()).toBeCloseTo(18.51);
      expect(result.toMinorUnits()).toBe(1851);
    });
  });

  describe("equalsTo", () => {
    it("returns true when the amounts match", () => {
      const value = createMoney(50);

      expect(value.equalsTo(createMoney(50))).toBe(true);
      expect(value.equalsTo(createMoney(40))).toBe(false);
    });
  });
});
```

**Characteristics:**
- Test mathematical operations directly without mocking
- Use `toBeCloseTo()` for floating-point comparisons
- Use factory helpers for consistent test data
- Multiple assertions per operation to verify state

### Service Testing

**Pattern for service tests:**

```typescript
describe("[ServiceName]", () => {
  let service: [ServiceName];
  let dependencyA: jest.Mocked<DependencyA>;
  let dependencyB: jest.Mocked<DependencyB>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        [ServiceName],
        {
          provide: DependencyA,
          useValue: {
            methodA: jest.fn(),
          },
        },
        {
          provide: DependencyB,
          useValue: {
            methodB: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<[ServiceName]>([ServiceName]);
    dependencyA = module.get(DependencyA);
    dependencyB = module.get(DependencyB);
  });

  describe("[methodName]", () => {
    it("should perform operation successfully", async () => {
      // Arrange
      const input = { /* test data */ };
      dependencyA.methodA.mockResolvedValue({ /* expected result */ });

      // Act
      const result = await service.methodName(input);

      // Assert
      expect(dependencyA.methodA).toHaveBeenCalledWith(
        expect.objectContaining(expectedCall),
      );
      expect(result).toEqual(expectedResult);
    });

    it("should propagate error from dependency", async () => {
      // Arrange
      const error = new Error("Dependency failed");
      dependencyA.methodA.mockRejectedValue(error);

      // Act & Assert
      await expect(service.methodName(input)).rejects.toThrow(error);
    });
  });
});
```

### Repository Testing

**Pattern for repository tests:**

From `src/business/test/repositories/business.repository.spec.ts`:

```typescript
describe("BusinessRepository", () => {
  let repository: BusinessRepository;
  let mongoConnection: jest.Mocked<MongoConnectionService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BusinessRepository,
        {
          provide: MongoConnectionService,
          useValue: {
            getCollection: jest.fn(),
          },
        },
      ],
    }).compile();

    repository = module.get<BusinessRepository>(BusinessRepository);
    mongoConnection = module.get(MongoConnectionService);
  });

  describe("create", () => {
    it("should insert and return business with ID", async () => {
      // Arrange
      const businessData = BusinessMockGenerator.createBusinessDto({
        id: undefined,
      });
      const mockCollection = {
        insertOne: jest.fn().mockResolvedValue({ insertedId: "new-id" }),
      };
      mongoConnection.getCollection.mockReturnValue(mockCollection);

      // Act
      const result = await repository.create(businessData);

      // Assert
      expect(mockCollection.insertOne).toHaveBeenCalledWith(
        expect.objectContaining(businessData),
      );
      expect(result.id).toBe("new-id");
    });
  });
});
```

### Coverage

**Targets:** Coverage collected via Jest

**View Coverage:**
```bash
npm run test              # Generates coverage report in coverage/ directory
```

**Coverage output location:** `coverage/` directory (collected by Jest configuration)

**Coverage collection from `package.json`:**
```json
{
  "collectCoverageFrom": ["**/*.(t|j)s"],
  "coverageDirectory": "../coverage"
}
```

### Error Handling Tests

**Pattern for testing error cases:**

```typescript
it("should throw InvalidRequestError when validation fails", async () => {
  // Arrange
  const invalidInput = { /* invalid data */ };

  // Act & Assert
  await expect(service.create(invalidInput)).rejects.toThrow(
    InvalidRequestError,
  );
});

it("should propagate error when access control fails", async () => {
  // Arrange
  const accessError = new Error("Access denied");
  authorizedCreator.mockRejectedValue(accessError);

  // Act & Assert
  await expect(service.create(user, data)).rejects.toThrow(accessError);
  expect(someRepository.create).not.toHaveBeenCalled();
});

it("should handle multiple error scenarios", async () => {
  // Arrange & Act & Assert - test different error paths
  const scenarios = [
    { input: null, expectedError: InvalidRequestError },
    { input: duplicateData, expectedError: DuplicateKeyError },
    { input: largeData, expectedError: ValidationError },
  ];

  for (const scenario of scenarios) {
    await expect(service.create(scenario.input)).rejects.toThrow(
      scenario.expectedError,
    );
  }
});
```

### Async Testing

**Pattern for async operations:**

```typescript
it("should handle async operations correctly", async () => {
  // Arrange
  const mockPromise = Promise.resolve(expectedResult);
  dependency.asyncMethod.mockReturnValue(mockPromise);

  // Act
  const result = await service.method();

  // Assert
  expect(dependency.asyncMethod).toHaveBeenCalled();
  expect(result).toEqual(expectedResult);
});

it("should reject on async error", async () => {
  // Arrange
  const error = new Error("Async failed");
  dependency.asyncMethod.mockRejectedValue(error);

  // Act & Assert
  await expect(service.method()).rejects.toThrow("Async failed");
});
```

### Test Data and Fixtures

**Location:** `test/mocks/` directories within each module

**Pattern:** Static generator classes with factory methods

Available generators:
- `BusinessMockGenerator` in `src/business/test/mocks/`
- `UserMockGenerator` in `src/user/test/mocks/`
- `CustomerMockGenerator` in `src/customer/test/mocks/`

**Usage in tests:**
```typescript
const business = BusinessMockGenerator.createBusinessDto({
  name: "Custom Name",
  status: BusinessStatus.INACTIVE,
});

const user = BusinessMockGenerator.createUserDto({
  email: "custom@example.com",
});
```

---

## Frontend Testing

**Status:** No tests configured

The frontend project (`trade-flow-ui`) does not have testing infrastructure set up. Vitest or Jest could be added if testing is required, but there is no existing test configuration or test files in the codebase.

---

## Testing Best Practices

### Test Organization

1. **One test file per source file** → `service.ts` → `service.spec.ts`
2. **Group tests with describe blocks** → Separate tests by method/feature
3. **Clear test names** → What should happen and under what condition
4. **Consistent Arrange-Act-Assert structure** → Easy to read and maintain

### Mock Management

1. **Mock at service boundaries** → Database, external APIs, third-party services
2. **Use type-safe mocks** → `jest.Mocked<T>` for proper typing
3. **Clear mock setup** → Each test specifies expected behavior
4. **Reset mocks after tests** → `jest.clearAllMocks()` in `afterEach()`

### Assertion Patterns

1. **Use `expect().toEqual()` for deep equality** → Objects with specific properties
2. **Use `expect().toBe()` for strict equality** → Primitives and references
3. **Use `expect.objectContaining()` for partial matching** → Only assert relevant fields
4. **Use `expect.any(Type)` for type checking** → Don't assert specific values
5. **Use `toHaveBeenCalledWith()` for mock verification** → Assert calls with proper arguments

### Examples from Codebase

**Service test with multiple error scenarios:**
From `src/business/test/services/business-creator.service.spec.ts`:
- Success case with happy path
- Error propagation from access control
- Error propagation from repository
- Error propagation from business user creation
- Error propagation from role assignment

**Value object test with factory helper:**
From `src/core/test/value-objects/money.value-object.spec.ts`:
- Helper function `createMoney()` for test data
- Multiple describe blocks for different operations
- Floating-point comparisons with `toBeCloseTo()`
- Comprehensive operator testing (add, subtract, multiply, divide, etc.)

---

*Testing analysis: 2026-02-21*
