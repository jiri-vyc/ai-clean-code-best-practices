# The Ultimate Best Practices for TypeScript Code
## For humans and agents alike

A practical guide for writing TypeScript that stays readable, testable, extensible, and safe to change.

This guide is inspired by:

- *Clean Code* by Robert C. Martin
- *Clean Architecture* by Robert C. Martin
- `clean-code-javascript` by Ryan McDermott
- `nodebestpractices` by Yoni Goldberg

It is intentionally opinionated for modern TypeScript projects where multiple contributors and AI tools work in the same codebase.

## Why this matters

Complexity compounds. When code becomes hard to read, changes start breaking unrelated behavior, debugging expands across too many files, and both humans and agents begin guessing.

This guide optimizes for three things:

- reduce cognitive and context load
- keep changes local
- make behavior inferable from names, types, and structure

## Who this guide is for

Use this guide for TypeScript projects that:

- are expected to grow
- have multiple contributors
- require safe, frequent change
- use AI-assisted development

It matters most when the codebase changes quickly and contributors need to make reliable edits without loading the whole system into context.

## The prime directive

### Readability first

Prefer code that is obvious over code that is clever.

Choose:

- readable over compact
- explicit over magical
- boring over surprising
- simple over over-abstracted
- local reasoning over cross-file guessing

Performance matters, but only after correctness and readability are secure.

### Optimize for small-context understanding

A reader should be able to answer these quickly:

- What does this do?
- Why does it exist?
- What can it change?
- What can it return or throw?
- Where do I modify this behavior?

If those answers require opening many files, the code is too expensive to work with.

## The non-negotiables

- Names must be meaningful, consistent, and searchable.
- Side effects and mutation must be explicit.
- Use strict typing. Avoid `any`, unsafe `as`, and stringly typed contracts.
- Keep one use case together. A feature or bug fix should touch a small, predictable set of files.
- Prefer some duplication over the wrong abstraction. Extract on the third clear repetition.
- Depend on contracts, not frameworks or transport details.
- Keep domain, application, infrastructure, and delivery concerns separate.
- Prefer guard clauses over deep nesting.
- Test failures and edge cases, not just success paths.

## Naming rules

Naming is one of the highest-leverage improvements you can make. Good names reduce the need to inspect implementation.

### Prefer intention-revealing names

```ts
// Bad
const data = await repo.get();

// Better
const activeSubscription = await subscriptionRepository.getActiveByUserId(userId);
```

```ts
// Bad
function handle(input: Input): Output

// Better
function calculateInvoiceTotals(input: InvoicePricingInput): InvoiceTotals
```

### Use strong verbs for functions

Prefer:

- `get` for cheap or in-memory retrieval
- `find` for maybe-present results
- `load` for external or heavier retrieval
- `create`, `update`, `delete`, `remove` for persistence changes
- `calculate`, `parse`, `map`, `validate`, `assert` for clear intent

### Encode side effects and mutation in names

Prefer names like:

- `loadUserById`
- `applyPreferencesPatch`
- `persistOrder`
- `publishEvent`
- `markInvoiceAsPaid`

Avoid names that sound observational for mutating behavior.

### Keep vocabulary consistent

Do not alternate between equivalent terms unless the domain truly distinguishes them.

Avoid unnecessary variation such as:

- `user`, `customer`, `accountHolder`
- `fetch`, `get`, `resolve`
- `remove`, `delete`, `clear`

### Avoid filler and shorthand names

Avoid names like `data`, `info`, `handler`, `process`, `obj`, `arr`, or one-letter variables even in trivial local scopes. Shorter names provide no benefit (saving on character count is useless).

### Booleans should read like predicates

Prefer:

- `isActive`
- `hasAccess`
- `canRetry`

Avoid unclear names like `active`, `access`, or `retry`.

### Prefer enums or literal unions over scattered strings

```ts
export enum PaymentStatus {
  Pending = "Pending",
  Authorized = "Authorized",
  Settled = "Settled",
  Failed = "Failed",
}
```

## Function design rules

### A function should do one thing

Ask:

> Can this function be summarized in one precise sentence?

If not, it probably does too much. Do not over-concern yourself about the function *length*, but rather its (single) responsibility.

### Prefer one level of abstraction per function

Do not mix orchestration with low-level details.

```ts
// Bad
async function registerUser(input: RegisterUserInput): Promise<UserId> {
  if (!EMAIL_REGEX.test(input.email)) {
    throw new InvalidEmailError(input.email);
  }

  const existingUser = await db.user.findUnique({ where: { email: input.email } });
  if (existingUser) {
    throw new UserAlreadyExistsError(input.email);
  }

  const hashedPassword = await bcrypt.hash(input.password, 12);

  const createdUser = await db.user.create({
    data: {
      email: input.email,
      passwordHash: hashedPassword,
    },
  });

  await emailClient.send({
    to: input.email,
    template: "welcome",
  });

  return createdUser.id;
}
```

```ts
// Better
async function registerUser(command: RegisterUserCommand): Promise<UserId> {
  assertValidEmail(command.email);
  await assertUserDoesNotExist(command.email);

  const passwordHash = await passwordHasher.hash(command.password);
  const createdUser = await userRepository.create({
    email: command.email,
    passwordHash,
  });

  await welcomeEmailSender.sendTo(createdUser.email);

  return createdUser.id;
}
```

### Prefer named parameters

When a function takes more than two parameters, prefer an object parameter. Prevents parameter-order errors, provides cleaner contract, is well-readable and usable.

```ts
// Bad
scheduleInvoice(userId, startDate, endDate, retryCount, priority);

// Better
scheduleInvoice({
  userId,
  startDate,
  endDate,
  retryCount,
  priority,
});
```

### Do not use boolean flags to choose behavior

```ts
// Bad
createReport(userId, true);

// Better
createDraftReport(userId);
createPublishedReport(userId);
```

### Prefer return early

```ts
// Bad
function getDiscount(order: Order): number {
  if (order.items.length > 0) {
    if (order.customer.isActive) {
      if (!order.customer.isBlocked) {
        return 0.1;
      }
    }
  }

  return 0;
}
```

```ts
// Better
function getDiscount(order: Order): number {
  if (order.items.length === 0) {
    return 0;
  }

  if (!order.customer.isActive) {
    return 0;
  }

  if (order.customer.isBlocked) {
    return 0;
  }

  return 0.1;
}
```

### Keep side effects isolated

Separate pure decision logic from impure orchestration.

### Avoid hidden mutation

Do not mutate parameters unless the API is explicitly mutating. If mutation is necessary, make it obvious in the name, for example `normalizeUserInPlace`.

### Keep output contracts stable

Do not mix `null`, thrown errors, partial returns, and log-and-swallow behavior in the same function contract.

Examples:

- `findUserById` returns `User | null`
- `getUserById` returns `User` or throws `UserNotFoundError`
- `parseInvoiceCsv` returns `ParseResult<InvoiceCsv>`

## TypeScript-specific rules

### Enable strict mode

At minimum:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true
  }
}
```

### Prefer precise types

- avoid `any`
- prefer `unknown` plus narrowing when the shape is not known
- avoid `as` casts when a better model or validation can express the truth

### Model state honestly

- avoid nested ternaries for non-trivial branching
- do not use optional fields when a value is required in a specific state
- make invalid states unrepresentable where practical
- use exhaustive checks for unions

```ts
function mapPaymentResult(result: PaymentResult): string {
  switch (result.status) {
    case PaymentStatus.Authorized:
      return "authorized";
    case PaymentStatus.Declined:
      return "declined";
    case PaymentStatus.Failed:
      return "failed";
    default:
      return assertUnreachable(result);
  }
}

function assertUnreachable(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}
```

### Keep third-party types at the edges

Translate framework, ORM, SDK, and transport types at the boundary instead of letting them leak into core business logic.

## Data, models, and boundaries

- Validate external data at entry points.
- Map between transport, persistence, and domain shapes once per boundary.
- Keep DTOs as transport contracts, not domain objects.

## Clean architecture for TypeScript

### The goal

Business rules should stay independent of frameworks, UI, databases, queues, HTTP, SDKs, and deployment details.

### The dependency rule

Dependencies point inward:

- outer layers may depend on inner layers
- inner layers must not depend on outer layers

In practice:

- domain should not know Express, React, Prisma, Stripe, Redis, or similar tools
- application coordinates use cases and depends on abstractions
- infrastructure implements those abstractions
- delivery adapts external input to application commands and maps results back out

### Recommended layer model

- Domain: entities, value objects, domain services, core business rules, business logic
- Application: use cases, commands, queries, application services, dependency interfaces, orchestration
- Infrastructure: repositories, persistence, SDK integrations, gateways, adapters
- Delivery (API layer): controllers, resolvers, CLI commands, workers, event consumers

### Architectural rule of thumb

- if a business rule changes, most edits should stay in domain or application, and within a bounded module
- if a transport changes, most edits should stay in delivery
- if a provider changes, most edits should stay in infrastructure

## Dependency injection and testability

### Depend on abstractions

Use cases should receive dependencies through constructor injection or function injection.

### Use a DI container when it helps composition

Use a container when dependency wiring becomes large, repetitive, environment-specific, or lifecycle-sensitive. Do not introduce one by default for small projects.

### Design for testing without spy-heavy mocking

Prefer pure domain logic where possible. When dependencies are needed, inject them from outside instead of reaching into globals or hidden singletons.

## Error handling

### Separate expected failures from bugs

Expected failures include domain and application outcomes such as duplicate email, insufficient balance, or forbidden operations.

Bugs include impossible branches, invariant violations, and broken assumptions.

Do not handle these the same way.

### Handle errors centrally at entry points

Controllers, workers, and CLI runners should translate application failures into transport-level responses.

Examples:

- domain error -> HTTP 409
- validation error -> HTTP 400
- unexpected error -> HTTP 500 with logging and monitoring

### Error handling rules

- use specific error types or result types
- never swallow errors silently
- preserve useful context when rethrowing
- validate configuration early and fail fast

## Testing strategy

### Test behavior, not implementation trivia

A good test verifies:

- given what input
- under what conditions
- what observable outcome occurs

Avoid tests that depend on private method ordering or non-essential helper calls.

### Cover the major outcome classes

For each use case, try to cover:

1. success
2. validation failure
3. business failure
4. infrastructure failure
5. unexpected error propagation or handling

### Use red-green testing

1. Write a failing test that captures the intended behavior.
2. Implement the change until the test passes.
3. Refactor if needed.

Do not edit the test just to force a pass.

### Test error flows deliberately

Examples:

- repository throws
- gateway times out
- config is invalid
- dependency returns malformed response
- retry path is exhausted
- event publish fails after state change

### Keep tests readable

Use clear names and obvious structure:

```ts
it("creates a user when the email is unique", ...)
it("rejects registration when the email already exists", ...)
```

```ts
// Arrange
const repository = new InMemoryUserRepository();
const useCase = new RegisterUser(repository, hasher, emailSender);

// Act
const userId = await useCase.execute(command);

// Assert
expect(userId).toBeDefined();
```

## Comments and documentation

### Prefer code that explains itself

The best comment is often a better name or a better function boundary.

### Write comments for why

Comments should be durable and explain things the code alone cannot, such as:

- non-obvious business rules
- unusual decisions
- protocol constraints
- workarounds with context
- performance tradeoffs

Avoid comments that merely restate the code or comments tied only to a temporary task. Write them so they stay relevant for the future. Do not write comments specific to current goal, or past behaviour that will NOT stay in the codebase.

### Keep README guidance operational

A project README should help contributors answer:

- how the project is structured
- where and how to add a feature
- how architecture works
- how to run tests and linting
- what conventions are mandatory

## Formatting and code style

### Use automated tools

At minimum:

- TypeScript compiler in strict mode
- ESLint
- Prettier

### Formatting rules

- prefer one statement per line
- keep functions visually scannable
- use whitespace to separate ideas
- avoid long chained expressions when intermediate names help
- avoid dense anonymous callback pyramids
- keep imports ordered consistently

## Final reminders

- Prefer readable code over clever code.
- Keep changes local and module boundaries clear.
- Let names, types, and structure carry most of the explanation.
- Contributor should be able to answer
    - where to look
    - where to change
    - what to trust
    - what to test
    
    just from the business language and without opening twenty files.
- The real benchmark is whether a new contributor can make a safe change with a small, localized diff.
