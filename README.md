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

The goal of this guide is simple: Optimize specifically for minimizing this effect.

That can be dissected into specific sub-goals:
- reduce cognitive/context load
- keep changes local
- make behavior inferable from names, types, and structure

## Table of Contents

1. [Who this guide is for](#who-this-guide-is-for)
2. [The prime directive](#the-prime-directive)
3. [The non-negotiables](#the-non-negotiables)
4. [Naming rules](#naming-rules)
5. [Function design rules](#function-design-rules)
6. [TypeScript-specific rules](#typescript-specific-rules)
7. [Data, models, and boundaries](#data-models-and-boundaries)
8. [Clean architecture for TypeScript](#clean-architecture-for-typescript)
9. [Dependency injection and testability](#dependency-injection-and-testability)
10. [Error handling](#error-handling)
11. [Testing strategy](#testing-strategy)
12. [Comments and documentation](#comments-and-documentation)
13. [Formatting and code style](#formatting-and-code-style)
14. [Final reminders](#final-reminders)

---

## Who this guide is for

Use this guide for TypeScript projects that:

- are expected to grow
- have multiple contributors
- require safe, frequent change
- use AI-assisted development

It matters most when the codebase changes quickly and new contributors need to make reliable edits without loading the whole system into context.

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

These are the default rules for this guide.

### 1. Names must be meaningful and consistent

Names should reveal:

- what a value represents
- what a function does
- what a result means
- whether something mutates state
- whether something is optional, cached, raw, validated, or persisted

Use one name per concept across the codebase.

### 2. Names must be searchable

Avoid vague one-letter names, overloaded words, and near-duplicates. Prefer names that are easy to grep, log, and review.

### 3. Side effects must be explicit

If a function mutates state, writes to disk, sends a request, emits an event, or updates external state, its name and placement should make that obvious.

### 4. Use strict typing everywhere

- No `any`
- No unsafe `as` assertions
- No hidden shape assumptions
- No stringly typed contracts when a type can model them

When the type system fights you, fix the model instead of bypassing it.

### 5. Keep one use case together

Code for one business capability should live near itself. A feature or bug fix should ideally touch a small, predictable set of files.

### 6. Avoid duplication, but do not worship DRY

Some duplication is cheaper than the wrong abstraction. Extract on the third clear repetition, not the first hint of similarity.

### 7. Prefer dependency inversion

Business logic should depend on contracts, not frameworks, databases, SDKs, or transport details.

### 8. Separate layers clearly

Keep domain rules, application orchestration, delivery, and infrastructure from bleeding into one another. Keep explicit module boundaries, focus on clean contracts between modules and layers.

### 9. Return early

Guard clauses reduce nesting and make the happy path visible.

### 10. Test unhappy paths too

Success-only testing is not enough. Cover invalid input, failures, rejections, and dependency problems. Use red-green testing.

---

## Naming rules

Naming is one of the highest-leverage improvements you can make. Good names reduce the need to inspect implementation.

### General naming principles

#### Prefer intention-revealing names

```ts
// Bad
const data = await repo.get();

// Better
const activeSubscription = await subscriptionRepository.getActiveByUserId(userId);
```

#### Name by meaning, not implementation detail

```ts
// Bad
function handle(input: Input): Output

// Better
function calculateInvoiceTotals(input: InvoicePricingInput): InvoiceTotals
```

#### Name values by what they hold

```ts
// Bad
const value = response.items[0];

// Better
const firstMatchingOrder = response.items[0];
```

#### Name functions by what they do

Use strong verbs:

- `get` for cheap or in-memory retrieval
- `find` for maybe-present results
- `load` for external or heavier retrieval
- `create`, `update`, `delete`, `remove` for persistence changes
- `calculate`, `parse`, `map`, `validate`, `assert` for clear intent

#### Encode side effects and mutation in names

Prefer names like:

- `loadUserById`
- `applyPreferencesPatch`
- `persistOrder`
- `publishEvent`
- `markInvoiceAsPaid`

Avoid names that sound observational for mutating behavior.

Never use meaning-less names like `data`, `info`, `handler`, `process` (they often signal weak boundaries or missing domain language) or one- or two-letter variable names. Even function parameters, anonymous function parameters.  Never shorthand unnecessarily (like `obj`, `arr`). Shorter names provide no benefit (saving on character count is useless).

#### Keep vocabulary consistent

Do not alternate between equivalent terms unless the domain truly distinguishes them.

Avoid unnecessary variation such as:

- `user`, `customer`, `accountHolder`
- `fetch`, `get`, `resolve`
- `remove`, `delete`, `clear`

#### Use searchable names

```ts
// Bad
const e = new Error("...");
const arr = build();
const obj = parse();

// Better
const paymentAuthorizationError = new PaymentAuthorizationError(details);
const eligibleInvoices = buildEligibleInvoices(query);
const parsedWebhookPayload = parseWebhookPayload(requestBody);
```

### Boolean naming rules

Booleans should read like predicates:

- `isActive`
- `hasAccess`
- `canRetry`

Avoid unclear names like `active`, `access`, or `retry`.

### Enum naming rules

Replace scattered raw string identifiers with enums or literal unions where appropriate.

```ts
export enum PaymentStatus {
  Pending = "Pending",
  Authorized = "Authorized",
  Settled = "Settled",
  Failed = "Failed",
}
```

---

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

### Use named parameters

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

Separate pure decision logic from impure orchestration. Prefer pure functions.

```ts
function calculateRefundAmount(input: RefundCalculationInput): Money {
  // Pure logic
}

async function processRefund(command: ProcessRefundCommand): Promise<void> {
  const refundAmount = calculateRefundAmount(command);
  await paymentGateway.refund({
    paymentId: command.paymentId,
    amount: refundAmount,
  });
}
```

### Avoid hidden mutation

Do not mutate parameters unless the API is explicitly mutating.

```ts
// Bad
function normalizeUser(user: User): void {
  user.email = user.email.trim().toLowerCase();
}

// Better
function normalizeUser(user: User): User {
  return {
    ...user,
    email: user.email.trim().toLowerCase(),
  };
}
```

If mutation is necessary, say so in the name:

```ts
function normalizeUserInPlace(user: MutableUser): void {
  user.email = user.email.trim().toLowerCase();
}
```

### Keep output contracts stable

A function should not sometimes return a value, sometimes `null`, sometimes throw, and sometimes log-and-swallow. Pick one contract and keep it.

Examples:

- `findUserById` returns `User | null`
- `getUserById` returns `User` or throws `UserNotFoundError`
- `parseInvoiceCsv` returns `ParseResult<InvoiceCsv>`

---

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

### No `any`

Use precise types. If the shape is unknown, model it as `unknown` and narrow it.

### Avoid `as` casts

Prefer narrowing, validation, and better type models over assertions.

### No nested ternaries

### Model optionality honestly

Do not use optional fields when the value is required in a specific state. Prefer separate types for separate lifecycle stages.

### Make invalid states unrepresentable where practical

If a command requires an authenticated user, accept `authenticatedUser: AuthenticatedUser`, not `user?: User`.

### Use exhaustive checks

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

---

## Data, models, and boundaries

### Validate at entry points

Validate external data at the boundary. Do not push untrusted input deep into the system.

### Map once per boundary

Convert between transport, persistence, and domain shapes once, in a predictable place.

### Keep DTOs dumb

DTOs are transport contracts, not domain objects.

---

## Clean architecture for TypeScript

### The goal

Business rules should stay independent of:

- frameworks
- UI
- databases
- queues
- HTTP
- SDKs
- deployment details

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

#### 1. Domain

Contains:

- entities
- value objects
- domain services
- core business rules
- all domain logic

Should know nothing about frameworks, databases, or transport.

#### 2. Application

Contains:

- use cases
- commands and queries
- application services
- interfaces for required dependencies

Depends on domain and abstract ports only.

#### 3. Infrastructure

Contains:

- repositories
- database access
- SDK integrations
- email gateways
- file storage adapters
- queue integrations
- logging implementations

Implements ports declared by inner layers.

#### 4. Delivery / Interface adapters / API layer

Contains:

- HTTP controllers
- GraphQL resolvers
- CLI commands
- worker handlers
- event consumers

Responsible for parsing input, auth/context extraction, validation, mapping, and response translation.

### Architectural rule of thumb

- if a business rule changes, most edits should stay in domain or application, within its module
- if a transport changes, most edits should stay in delivery
- if a provider changes, most edits should stay in infrastructure

---

## Dependency injection and testability

### Depend on abstractions

Use cases should receive dependencies through constructor injection or function injection.

```ts
export class RegisterUser {
  public constructor(
    private readonly userRepository: UserRepository,
    private readonly passwordHasher: PasswordHasher,
    private readonly welcomeEmailSender: WelcomeEmailSender,
  ) {}

  public async execute(command: RegisterUserCommand): Promise<UserId> {
    // ...
  }
}
```

### Use a DI container when it helps composition

Use a container when dependency wiring becomes large, repetitive, environment-specific, or lifecycle-sensitive. Do not introduce one by default for small projects.

### Design for testing without spy-heavy mocking

Prefer pure domain logic where possible. When dependencies are needed, inject them from outside instead of reaching into globals or hidden singletons.

---

## Error handling

### Separate expected failures from bugs

Expected failures include domain and application outcomes such as duplicate email, insufficient balance, or forbidden operations.

Bugs include impossible branches, invariant violations, and broken assumptions.

Do not handle these the same way.

### Use specific error types or result types

```ts
export class UserAlreadyExistsError extends Error {
  public constructor(public readonly email: Email) {
    super(`User already exists for email ${email}`);
  }
}
```

### Handle errors centrally at entry points

Controllers, workers, and CLI runners should translate application failures into transport-level responses.

Examples:

- domain error -> HTTP 409
- validation error -> HTTP 400
- unexpected error -> HTTP 500 with logging and monitoring

### Never swallow errors silently

Either handle the error meaningfully, enrich and rethrow it, or translate it into a typed failure.

### Preserve useful context

```ts
throw new PaymentCaptureError(
  `Failed to capture payment ${paymentId}`,
  { cause: error },
);
```

### Validate configuration early

Fail fast on invalid environment or configuration instead of allowing broken setup to surface during runtime.

## Testing strategy

### Test behavior, not implementation trivia

A good test verifies:

- given what input
- under what conditions
- what observable outcome occurs

Avoid tests that depend on private method ordering or non-essential helper calls.

### Test the major outcome classes

For each use case, try to cover:

1. success
2. validation failure
3. business failure
4. infrastructure failure
5. unexpected error propagation or handling

### Use red-green testing approach

1. Write a minimal failing test(s) that captures the indented behaviour, where current code fails (RED)
2. Implement the changes in code so that the test passes (GREEN)
3. Refactor and cleanup if needed

Do not edit test just to pass at any cost.

### Test error flows deliberately

Examples:

- repository throws
- gateway times out
- config is invalid
- dependency returns malformed response
- retry path is exhausted
- event publish fails after state change

### Name tests clearly

```ts
it("creates a user when the email is unique", ...)
it("rejects registration when the email already exists", ...)
it("returns 409 when the use case reports duplicate email", ...)
```

### Arrange, act, assert

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

### Write durable comments

If you write comments, write them so they STAY RELEVANT for the future. Do not write comments specific to current goal, or past behaviour that will NOT stay in the codebase. Treat comments as long-lived artifacts beside the current code. Explain and provide important context.

### Write comments for why, not what

Useful comments explain:

- a non-obvious business rule
- why an unusual decision exists
- protocol constraints
- workarounds with context
- performance tradeoffs

Avoid comments that merely restate the code.

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
- avoid nested ternaries
- avoid dense anonymous callback pyramids
- keep imports ordered consistently

## Final reminders

### Prefer readable code

Boring readable code scales better than clever code.

### Make change local

A clean codebase lets a contributor answer:

- where to look
- where to change
- what to trust
- what to test

without opening twenty files.

### Let the code explain itself

Good names, strong types, focused separated modules, explicit boundaries, and focused functions reduce the need for extra explanation. AI contributor can scan codebase quickly and find entry-points reliably just by inferring from the requirement's business language. Focus on ubiqutous language.

### The ultimate standard

A codebase is healthy when a new contributor can safely make a change with confidence and a small, localized diff.
