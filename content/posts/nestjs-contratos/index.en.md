---
title: Using contracts with NestJS, facilitating your unit tests
date: 2023-10-11T14:56:59-03:00
draft: false
tableOfContents: false
---

## What are contracts?

Contracts are a way of separating responsibilities between the parties that use this implementation, contracts are widely used in the context of software development following the DDD (Domain-Driven Design) standard.

Let's say we have a service called `UserService`, this service needs to make a call to the database and fetch user data. For this we have a repository layer, which is responsible for communicating with the database. In this example, it would look like this:

`controller` > `service` > `repository`

Imagine that you are going to write a unit test for the service layer, but as it is a unit test, you should not call the repository and call the database, that is why the contract comes into play.

Would be like this:

`controller` > `service` > `contract` > `repository`

Let's go to the code:
In the example below we have a simple service, which searches for the user by id, directly calling our repository.

```typescript
@Injectable()
export class UserService {
 constructor(private repository: UserRepository) {}

  async findUserById(id: string): Promise<User | null> {
    const findUser = await this.repository.findById(id)
    if !findUser throw new NotFoundException("user not found")
  }
}
```

Below is our repository, in the example we use the Prisma ORM, but it could be any other. We search for the user by ID, if we don't find it we return `null`, if not, we return the user.

```typescript
@Injectable()
export class UserRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    const findUser = await this.prisma.findUnique({
      where: { id },
    });

    if (!findUser) return null;

    return findUser;
  }
}
```

Now, imagine doing the unit test of our service, you would need to make mocks of your repository, but it depends on the connection with Prisma, so you would need to create a mock of that connection too, this makes the test more time consuming and tiring. But it can be resolved with contracts.

Let's go, below we create a contract using abstract class:

```typescript
export abstract class UserContractRepository {
  abstract findById(id: string): Promise<User | null>;
}
```

Now, just implement this class, so everyone who uses this contract needs to implement the `findById` method, let's change the service to use the contract:

```typescript
@Injectable()
export class UserService {
  constructor(private repository: UserContractRepository) {}

  async findUserById(id: string): Promise<User | null> {
    const findUser = await this.repository.findById(id);
    if (!findUser) throw new NotFoundException('user not found');
  }
}
```

In the service almost nothing is changed, instead of calling the repository directly, let's call the contract.

Let's change the repository:

```typescript
@Injectable()
export class UserRepository implements UserContractRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    const findUser = await this.prisma.user.findUnique({
      where: { id },
    });

    if (!findUser) return null;

    return findUser;
  }
}
```

In the repository, we now use implements, so the code editor already reports an error, if you don't have the complete implementation of `UserContractRepository`, in the example, we only have `findById` which receives a string per parameter and returns a `User` or `null`, so we implemented `UserContractRepository`, if you add more methods to `UserContractRepository`, you will need to add them to `UserRepository`.

```typescript
export abstract class UserContractRepository {
  abstract findById(id: string): Promise<User | null>;
  abstract createUser(user: User): Promise<void>;
}
```

Now, we have `createUser`, `UserRepository` should already report an error, as it does not fully implement `UserContractRepository`. Let's implement:

```typescript
@Injectable()
export class UserRepository implements UserContractRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    const findUser = await this.prisma.user.findUnique({
      where: { id },
    });

    if (!findUser) return null;

    return findUser;
  }

  async createUser(user: User): Promise<void> {
    const findUser = await this.prisma.user.create({
      data: user,
    });
  }
}
```

To work correctly in NestJs, we just need to change our module, so that when calling `UserContractRepository` the `UserRepository` class is used, it's very simple:

```typescript
@Module({
  providers: [
    {
      provide: UserContractRepository,
      useClass: UserRepository
    }
  ]
})
```

Well, what's the advantage of that?

Now as we have a layer between the service and repository, for unit tests we no longer need to call the repository directly, just create a fake implementation that implements the `UserContractRepository` and use it in the tests, and even, if we need to exchange the prism for another ORM , like TypeORM for example, we don't need to change our service, just create a new repository and implement `UserContractRepository`.

Let's make a fake repository, normally we call it in memory:

```typescript
export class UserRepositoryInMemory implements UserContractRepository {
  async findById(id: string): Promise<User | null> {
    return
  }

  async createUser(user: User): Promise<void> {
    return
}
```

## Returning a mock

In the example above, we didn't return anything, but you could return a mock of the User, like this:

```typescript
const userMock: User = {
  id: "id_mock",
  name: "John Doe",
  email: "john.doe@mail.com"
}

export class UserRepositoryInMemory implements UserContractRepository {
  async findById(id: string): Promise<User | null> {
    return userMock
  }

  async createUser(user: User): Promise<void> {
    return
}
```

We can further simulate the repository's behavior by checking if the user ID is the same as the `userMock`:

```typescript
const userMock: User = {
  id: "id_mock",
  name: "John Doe",
  email: "john.doe@mail.com"
}

export class UserRepositoryInMemory implements UserContractRepository {
  async findById(id: string): Promise<User | null> {
    if(id !== userMock.id) return null

    return userMock
  }

  async createUser(user: User): Promise<void> {
    return
}
```

Let's see how to do unit testing using Jest

```typescript
let sut: UserService;
let repository: UserContractRepository;

beforeEach(async () => {
  const module: TestingModule = await Test.createTestingModule({
    providers: [
      UserService,
      {
        provide: UserContractRepository,
        useClass: UserRepositoryInMemory,
      },
    ],
  }).compile();

  sut = module.get < UserService > UserService;
  repository = module.get < UserContractRepository > UserContractRepository;
});
```

We use the same logic used in the module, but now calling the repository in memory, this way we don't need to deal with the prism ORM implementation for example.

The test would look like this:

```typescript
describe('findUserById', () => {
  it('should be able to return user by id', async () => {
    const result = await sut.findUserById("id_mock");

    expect(result).toBeTruthy();
    expect(result).toHaveProperty('id');
    expect(result).toHaveProperty('name');
    expect(result).toHaveProperty('email');
  });
```

See how simple it is to test with contracts, you can also simulate an error, using jest's `spyOn`, for example:

```typescript
describe('findUserById', () => {
  it('should not be able to return user by id', async () => {
  jest.spyOn(repository, 'findById').mockResolvedValueOnce(null);

    expect(() => {
      return sut.findUserById("id_mock");
    }).rejects.toThrow(NotFoundException);
  });
```

But since we have our in memory, with the treatment to return null if the user ID is different, it would be enough to pass a wrong ID:

```typescript
describe('findUserById', () => {
  it('should not be able to return user by id', async () => {

    expect(() => {
      return sut.findUserById("id_mock_wrong");
    }).rejects.toThrow(NotFoundException);
  });
```

## Conclusion

Using contracts in NestJS can be a very effective approach to facilitate unit testing and improve code maintainability. By separating the repository interface into a contract, you create an intermediate layer between the service and the repository, allowing you to create alternative implementations for testing purposes, such as `UserRepositoryInMemory`. This has several advantages:

**1. Makes unit testing easier:** With the contract in place, you can create dummy repository implementations that behave as needed for each test, without needing to interact with the real database or persistence layer.

**2. Test isolation:** Isolating the service to be tested from its external dependencies, such as the ORM or other services, is fundamental to effective unit testing. Contracts help achieve this isolation.

**3. Flexibility:** If you decide to switch from one ORM to another, such as from Prisma to TypeORM, the services do not need to change. Just create a new repository that implements the existing contract and make the change in the module.

**4. Better adherence to DDD:** The contract approach aligns well with the principles of Domain-Driven Design (DDD), where emphasis is placed on the clarity of interfaces between different layers of the application.

**5. Simpler error tests:** You can easily simulate different error scenarios, such as returning “null” or throwing exceptions, to ensure that your service properly handles these situations.

Using contracts in NestJS is a best practice that can make your code more testable, more flexible, and more robust. This helps keep your code clean and makes it easier to evolve and maintain the system over time.
