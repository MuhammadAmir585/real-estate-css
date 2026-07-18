
# World-Class PHP Framework Creator Roadmap

You are not trying to write a collection of helper classes. You are trying to create:

1. A runtime architecture.
2. A stable programming model.
3. A package ecosystem.
4. A developer experience.
5. A security boundary.
6. A backward-compatibility promise.
7. An open-source community.

A framework becomes successful when developers can build reliable software **without understanding every internal mechanism**, while advanced developers can replace components without fighting hidden magic.

---

# 1. Recommended Technical Baseline

As of July 2026, PHP 8.5 is the newest major PHP branch. PHP 8.2 through 8.5 are supported, although PHP 8.2 is already in security-only support and reaches end of support in December 2026. PHPUnit 12 requires PHP 8.3 or later. ([PHP][1])

For a new framework, I recommend:

```text
Minimum PHP:      PHP 8.4
Primary runtime:  PHP 8.5
CI matrix:        PHP 8.4 and PHP 8.5
Experimental CI:  Next PHP development branch, allowed to fail
Composer:         Composer 2
Testing:          PHPUnit 12
Static analysis:  PHPStan level 10
Coding style:     PER Coding Style 3.0
```

PER Coding Style 3.0 now extends and replaces PSR-12, while PSR-2 is deprecated. ([PHP-FIG][2])

A minimum of PHP 8.4 gives you:

* Strong modern typing.
* Attributes.
* Enums.
* Readonly classes and properties.
* Constructor property promotion.
* Intersection and union types.
* First-class callable syntax.
* Modern reflection.
* Better error handling.
* A reasonable support lifetime.

Do not use every new language feature merely because it exists. Framework code should favor **predictable APIs over clever syntax**.

---

# 2. How Framework Creators Think

Application developers usually ask:

> How do I implement this feature?

Framework creators must ask:

> What general problem does this feature represent?

Then:

> What is the smallest stable abstraction that solves that class of problems?

And finally:

> Can developers replace my implementation without replacing the whole framework?

For every feature, answer these questions:

1. What behavior does it guarantee?
2. What behavior is deliberately unspecified?
3. Which part is public API?
4. Which part is internal implementation?
5. How can it be extended?
6. How can it be replaced?
7. What happens when it fails?
8. How will it behave in long-running workers?
9. How will it be tested?
10. How can it evolve without breaking users?

That is the difference between designing a framework and building an application.

---

# 3. Your Framework Philosophy

Before writing the container or router, create a one-page philosophy document.

A strong starting philosophy would be:

> The framework provides a small, explicit, standards-compatible application kernel. Features are supplied through replaceable packages. Defaults are productive, but application code is not forced to depend on framework-specific implementations.

## Core principles

### 1. Interoperability first

Use PHP-FIG interfaces at package boundaries wherever they accurately represent your requirements.

PHP-FIG currently lists accepted standards for logging, autoloading, caching, HTTP messages, containers, events, middleware, HTTP factories, HTTP clients and clocks. Draft standards should not be treated as stable framework contracts. ([PHP-FIG][3])

### 2. Small core

The core should not contain:

* An ORM.
* A template engine.
* A queue server.
* A mail transport.
* A filesystem implementation.
* A complete authentication system.
* Vendor-specific cloud integrations.

It should contain only the machinery required to boot and execute applications.

### 3. Explicit over magical

Magic is acceptable only when:

* It has deterministic rules.
* It can be inspected.
* It produces useful errors.
* It can be disabled.
* It is compiled or cached in production.
* The explicit alternative remains available.

### 4. Secure defaults

The easiest way to use the framework should also be the safest reasonable way.

### 5. Worker-safe by design

Do not assume that every request runs in a completely new process.

Avoid:

* Mutable global state.
* Static request objects.
* Request information stored in singletons.
* Pipeline indexes stored on shared objects.
* Services that retain one user’s data for the next request.

### 6. Progressive disclosure

A beginner should be able to write:

```php
$app->get('/users/{id}', ShowUser::class);
```

An advanced developer should be able to replace:

* The container.
* The router.
* The HTTP message implementation.
* The logger.
* The cache.
* The event dispatcher.
* The view renderer.
* The database layer.

---

# 4. Define What Your Framework Will Not Be

Write non-goals before implementation.

Example:

```text
The framework will not:

1. Copy Laravel’s API.
2. require an Active Record ORM.
3. require static facades.
4. require annotations or attributes.
5. hide the request-response lifecycle.
6. bundle every integration into the core.
7. invent replacements for mature PHP standards.
8. treat the service container as application architecture.
9. support unsupported PHP versions.
10. promise API stability before 1.0.
```

Non-goals prevent the project from becoming an unfocused Laravel/Symfony hybrid.

---

# 5. Recommended High-Level Architecture

Use a **PSR-15 middleware pipeline as the outer runtime**, with a small number of explicit lifecycle events.

Symfony’s HttpKernel uses an event dispatcher to structure the conversion of a request into a response. Slim organizes middleware as concentric layers. Mezzio is explicitly built around PSR-7, PSR-15 and PSR-11. Laravel exposes kernel operations for bootstrap, request handling and termination. ([Symfony][4])

Your design can take the strongest ideas from each without copying their APIs.

```text
                       APPLICATION CODE
                 Controllers / Domain / Modules
                              |
                              v
+-------------------------------------------------------------+
|                     APPLICATION DISTRIBUTION                |
|   Skeleton, default configuration, providers, CLI commands  |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                         HTTP KERNEL                         |
|                                                             |
|  Request Factory                                            |
|       |                                                     |
|       v                                                     |
|  Error Middleware                                           |
|       |                                                     |
|  Request Context / Trusted Proxy / Request ID                |
|       |                                                     |
|  Routing Middleware                                         |
|       |                                                     |
|  Session / Authentication / Authorization / CSRF             |
|       |                                                     |
|  Controller Dispatcher                                      |
|       |                                                     |
|  Response Finalizers                                        |
|       |                                                     |
|  Response Emitter                                           |
+-------------------------------------------------------------+
          |                |                 |
          v                v                 v
     Container          Events            Logger
       PSR-11           PSR-14            PSR-3
          |                |                 |
          +----------------+-----------------+
                           |
                           v
+-------------------------------------------------------------+
|                    OPTIONAL COMPONENTS                      |
|                                                             |
| Cache | Views | DBAL | Migrations | Queue | Mail | Storage   |
| Validation | Translation | Scheduler | Auth | Rate Limiting  |
+-------------------------------------------------------------+
```

---

# 6. The Core Dependency Rule

Your package graph should point inward.

```text
                    Optional Integrations
        Database / Queue / View / Mail / Session / Auth
                              |
                              v
                     Framework Components
           Router / Container / Events / Console / HTTP
                              |
                              v
                     Framework Contracts
                              |
                              v
                     PHP + PSR Interfaces
```

The contracts package must not depend on concrete framework implementations.

For example:

```text
framework/contracts
    depends on:
        php
        psr/container
        psr/http-message
        psr/http-server-handler
        psr/log

framework/kernel
    depends on:
        framework/contracts
        framework/http
        framework/routing

framework/database
    depends on:
        framework/contracts
        PDO
```

Never create a circular dependency such as:

```text
kernel -> router -> container -> kernel
```

---

# 7. Recommended Package Structure

Start development in a monorepo. Splitting twenty repositories before APIs stabilize creates release and dependency-management work without giving you architectural quality.

```text
framework/
├── composer.json
├── LICENSE
├── README.md
├── SECURITY.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
├── phpunit.xml
├── phpstan.neon
├── src/
│   ├── Contracts/
│   ├── Bootstrap/
│   ├── Container/
│   ├── Config/
│   ├── Http/
│   ├── Kernel/
│   ├── Middleware/
│   ├── Routing/
│   ├── Controller/
│   ├── Events/
│   ├── Error/
│   ├── Logging/
│   ├── Console/
│   └── Support/
├── tests/
│   ├── Unit/
│   ├── Contract/
│   ├── Integration/
│   ├── Functional/
│   └── Fixtures/
├── docs/
│   ├── architecture/
│   ├── concepts/
│   ├── tutorials/
│   ├── how-to/
│   ├── reference/
│   └── decisions/
├── examples/
│   ├── hello-world/
│   ├── rest-api/
│   └── modular-application/
└── tools/
```

Once boundaries become stable, publish packages such as:

```text
yourframework/contracts
yourframework/container
yourframework/http
yourframework/kernel
yourframework/router
yourframework/events
yourframework/console
yourframework/config
yourframework/cache
yourframework/validation
yourframework/session
yourframework/security
yourframework/database
yourframework/migrations
yourframework/queue
yourframework/view
yourframework/framework
```

The final `yourframework/framework` package becomes the convenient full-stack distribution.

Composer provides PSR-4, classmap and file autoloading, and published packages require properly structured `composer.json` metadata. Production autoloader optimization should be enabled during deployment rather than normal development. ([Composer][5])

---

# 8. Standards Adoption Strategy

Do not blindly put a PSR interface around everything. Use standards where they provide real interchangeability.

| Concern           | Standard             | Framework decision            |
| ----------------- | -------------------- | ----------------------------- |
| Autoloading       | PSR-4                | Required                      |
| Logging           | PSR-3                | Public logger dependency      |
| Cache             | PSR-6 / PSR-16       | Support both through adapters |
| HTTP messages     | PSR-7                | Public HTTP types             |
| Container         | PSR-11               | Public lookup interface       |
| Events            | PSR-14               | Event dispatcher boundary     |
| Server middleware | PSR-15               | Main HTTP pipeline            |
| HTTP factories    | PSR-17               | Response and stream creation  |
| HTTP client       | PSR-18               | External HTTP integrations    |
| Clock             | PSR-20               | Time-sensitive services       |
| Coding style      | PER Coding Style 3.0 | Repository standard           |

PSR-7 defines request, response, URI, stream and uploaded-file abstractions. PSR-15 defines request handlers and middleware. PSR-17 provides factories so reusable middleware does not have to depend on a particular PSR-7 implementation. ([PHP-FIG][6])

## Important distinction

Use PSR interfaces at **integration boundaries**.

Internally, you can use richer abstractions where necessary.

For example:

```php
interface RouterInterface
{
    public function match(
        ServerRequestInterface $request
    ): RouteMatch;
}
```

There is no need to create a fake standard merely to make your API look abstract.

---

# 9. Phase 0: Computer Science Foundations

Before building production components, study the ideas each component represents.

## Topics to master

### HTTP

Study:

* Request methods.
* Status codes.
* Headers.
* Content negotiation.
* Caching semantics.
* Conditional requests.
* Cookies.
* Proxies.
* Streaming.
* Uploads.
* Idempotency.
* Trusted proxy handling.

### Dependency graphs

Understand:

* Directed acyclic graphs.
* Topological ordering.
* Cycle detection.
* Lazy dependency resolution.
* Service lifetimes.
* Graph compilation.

### Routing algorithms

Study:

* Hash-table lookup for static routes.
* Regular-expression dispatch.
* Prefix trees and radix trees.
* Route precedence.
* Conflict detection.
* Method dispatch.
* URL generation.

### Language and runtime mechanics

Study:

* Reflection.
* Attributes.
* SPL interfaces.
* Exceptions and errors.
* Generators.
* Closures.
* Serialization limitations.
* Object lifetimes.
* PHP-FPM versus persistent workers.
* OPcache behavior.

### API design

Study:

* Liskov substitution.
* Interface segregation.
* Dependency inversion.
* Semantic versioning.
* Backward compatibility.
* Deprecation policies.
* Immutability.
* Command-query separation.

## Phase 0 deliverables

Do not proceed until you have written:

```text
docs/architecture/vision.md
docs/architecture/non-goals.md
docs/architecture/package-boundaries.md
docs/architecture/request-lifecycle.md
docs/architecture/backward-compatibility.md
docs/architecture/security-model.md
```

---

# 10. Phase 1: Build the HTTP Foundation

Your first executable goal should be:

> Convert a PHP server request into a PSR-7 response and emit it.

Do not begin with authentication, ORM or template rendering.

## Basic front controller

```php
<?php

declare(strict_types=1);

use Framework\Bootstrap\ApplicationFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$application = ApplicationFactory::fromBasePath(
    dirname(__DIR__)
)->createHttpApplication();

$request = $application
    ->serverRequestFactory()
    ->fromGlobals();

$response = $application->handle($request);

$application
    ->responseEmitter()
    ->emit($response);

$application->terminate($request, $response);
```

The front controller should contain almost no framework logic.

Its responsibilities are:

1. Load Composer.
2. Create the application.
3. Create the request.
4. Handle the request.
5. Emit the response.
6. Perform safe termination work.

---

# 11. HTTP Kernel Design

The kernel should be intentionally boring.

```php
<?php

declare(strict_types=1);

namespace Framework\Kernel;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;

final readonly class HttpKernel implements RequestHandlerInterface
{
    public function __construct(
        private RequestHandlerInterface $pipeline,
    ) {
    }

    public function handle(
        ServerRequestInterface $request
    ): ResponseInterface {
        return $this->pipeline->handle($request);
    }
}
```

The kernel is not where every feature belongs. It orchestrates the pipeline.

## Recommended lifecycle

```text
1. Bootstrap immutable configuration
2. Build or load compiled container
3. Create request
4. Assign request context
5. Execute middleware pipeline
6. Match route
7. Resolve controller
8. Invoke controller
9. Normalize result into response
10. Run response middleware
11. Emit response
12. Execute bounded termination hooks
13. Reset request-scoped services
```

---

# 12. Middleware Pipeline

A middleware pipeline gives you composable request processing.

```text
Request
   |
   v
Error Handler
   |
   v
Request ID
   |
   v
Trusted Proxy
   |
   v
Body Size Limit
   |
   v
Router
   |
   v
Session
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Controller
   |
   v
Response
```

## Worker-safe dispatcher

Do not store the current middleware index as mutable state on a singleton dispatcher.

This dangerous implementation can leak execution state:

```php
final class BadDispatcher
{
    private int $index = 0;
}
```

Instead, create an immutable next-handler for every dispatch operation.

```php
<?php

declare(strict_types=1);

namespace Framework\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

final readonly class MiddlewarePipeline implements RequestHandlerInterface
{
    /**
     * @param list<MiddlewareInterface> $middleware
     */
    public function __construct(
        private array $middleware,
        private RequestHandlerInterface $fallback,
    ) {
    }

    public function handle(
        ServerRequestInterface $request
    ): ResponseInterface {
        return $this->handlerAt(0)->handle($request);
    }

    private function handlerAt(int $index): RequestHandlerInterface
    {
        $middleware = $this->middleware[$index] ?? null;

        if ($middleware === null) {
            return $this->fallback;
        }

        return new NextMiddlewareHandler(
            middleware: $middleware,
            next: $this->handlerAt($index + 1),
        );
    }
}

final readonly class NextMiddlewareHandler implements RequestHandlerInterface
{
    public function __construct(
        private MiddlewareInterface $middleware,
        private RequestHandlerInterface $next,
    ) {
    }

    public function handle(
        ServerRequestInterface $request
    ): ResponseInterface {
        return $this->middleware->process(
            $request,
            $this->next,
        );
    }
}
```

A production implementation can prebuild the handler chain during application boot.

---

# 13. Build Your Own PSR-7 Implementation—or Not?

For learning, build one.

For framework adoption, do not force everyone to use it.

Recommended structure:

```text
yourframework/http-contracts
    PSR interfaces only

yourframework/http-message
    Your default PSR-7 implementation

yourframework/http-kernel
    Accepts any PSR-7 implementation
```

## What you will learn by implementing PSR-7

* Case-insensitive headers.
* Original header casing.
* Immutable `with*()` methods.
* Stream ownership.
* Seekable and non-seekable bodies.
* URI normalization.
* Uploaded-file movement.
* Request targets.
* Server parameters.
* Cookie and query parameters.

## Critical tests

Test that:

```php
$new = $response->withHeader('X-Test', 'value');

self::assertNotSame($response, $new);
self::assertFalse($response->hasHeader('X-Test'));
self::assertSame('value', $new->getHeaderLine('x-test'));
```

Your own implementation should be a default, not a prison.

---

# 14. Router Architecture

Separate these concerns:

```text
Route Definition
Route Collection
Route Compilation
Route Matching
Route Result
URL Generation
Route Cache
```

Do not create one 2,000-line `Router` class.

## Route definition

```php
final readonly class Route
{
    /**
     * @param non-empty-list<string> $methods
     * @param array<string, string> $requirements
     */
    public function __construct(
        public array $methods,
        public string $path,
        public mixed $handler,
        public ?string $name = null,
        public array $requirements = [],
        public ?string $host = null,
    ) {
    }
}
```

## Matching result

Do not return `null` for every routing failure.

```php
enum RouteMatchStatus
{
    case Found;
    case NotFound;
    case MethodNotAllowed;
}
```

```php
final readonly class RouteMatch
{
    /**
     * @param array<string, string> $parameters
     * @param list<string> $allowedMethods
     */
    public function __construct(
        public RouteMatchStatus $status,
        public ?Route $route = null,
        public array $parameters = [],
        public array $allowedMethods = [],
    ) {
    }
}
```

This lets the HTTP layer distinguish:

```text
404 Not Found
405 Method Not Allowed
Successful route match
```

## Router implementation stages

### Version 0.1

Use a simple ordered list and compiled regular expressions.

### Version 0.2

Separate static routes:

```php
$staticRoutes['GET']['/users'] = $route;
```

Dynamic routes go into compiled match groups.

### Version 0.3

Add:

* Named routes.
* URL generation.
* Host matching.
* Route groups.
* Prefixes.
* Middleware metadata.
* Requirement validation.
* Duplicate-route detection.

### Version 0.4

Add production route compilation:

```text
config/routes.php
      |
      v
RouteCompiler
      |
      v
var/cache/routes.php
```

The cached output should be ordinary executable PHP arrays, not serialized closures.

## Attributes

Keep attributes in an optional package:

```php
#[Get('/users/{id}', name: 'users.show')]
public function __invoke(string $id): ResponseInterface
{
}
```

Attribute scanning must compile into the same route representation as configuration-based routes.

The router must not care where routes came from.

---

# 15. Dependency Injection Container

The container is a composition tool, not your application architecture.

## Public interface

Expose PSR-11:

```php
interface ContainerInterface
{
    public function get(string $id): mixed;

    public function has(string $id): bool;
}
```

PSR-11 defines the common container lookup boundary. ([PHP-FIG][3])

## Internal container builder

Your richer API can be separate:

```php
$builder->bind(
    LoggerInterface::class,
    JsonLogger::class,
);

$builder->singleton(
    RouterInterface::class,
    CompiledRouter::class,
);

$builder->factory(
    RequestContext::class,
    static fn (): RequestContext => new RequestContext(),
);
```

## Service lifetimes

Support explicit lifetimes:

```text
Singleton:
    One instance for the application or worker.

Request scoped:
    One instance for the current HTTP request.

Transient:
    New instance every time.

Factory:
    User-defined creation logic.
```

Request scope is essential for persistent workers.

## Autowiring rules

A predictable autowiring algorithm:

```text
1. Is there an explicit definition?
2. Is there an alias?
3. Is the requested type instantiable?
4. Reflect on the constructor.
5. Resolve typed class dependencies.
6. Reject unresolved scalar parameters.
7. Reject ambiguous union types.
8. Detect circular dependencies.
9. Return a detailed dependency trace on failure.
```

Do not guess scalar values:

```php
final class ApiClient
{
    public function __construct(
        string $apiKey,
        int $timeout,
    ) {
    }
}
```

Require explicit configuration:

```php
$builder->factory(
    ApiClient::class,
    fn (ContainerInterface $container): ApiClient =>
        new ApiClient(
            apiKey: $container->get(Config::class)
                ->string('services.api.key'),
            timeout: $container->get(Config::class)
                ->int('services.api.timeout'),
        ),
);
```

## Circular dependency error

A useful error:

```text
Circular dependency detected:

UserService
 -> BillingService
 -> NotificationService
 -> UserService
```

A useless error:

```text
Maximum function nesting reached.
```

## Container compilation

Development mode:

```text
Reflection + runtime diagnostics
```

Production mode:

```text
Definition graph
      |
      v
Container compiler
      |
      v
Generated PHP factories
      |
      v
No repeated reflection
```

## Do not encourage service location

Avoid this:

```php
final class CheckoutService
{
    public function __construct(
        private ContainerInterface $container
    ) {
    }
}
```

Prefer:

```php
final class CheckoutService
{
    public function __construct(
        private PaymentGateway $payments,
        private OrderRepository $orders,
        private EventDispatcherInterface $events,
    ) {
    }
}
```

The container should normally appear only in:

* Bootstrap code.
* Factories.
* Providers.
* Framework dispatch code.

---

# 16. Configuration Architecture

Configuration should be immutable after application boot.

## Recommended precedence

```text
Framework defaults
        <
Package defaults
        <
Application configuration
        <
Environment-specific configuration
        <
Environment variables / secret provider
```

## Important rule

Read environment variables during bootstrap, then convert them into typed configuration.

Do not scatter this throughout application code:

```php
$host = getenv('DATABASE_HOST');
```

Use:

```php
$host = $config->string('database.host');
```

## Typed configuration

```php
interface Configuration
{
    public function has(string $key): bool;

    public function get(string $key): mixed;

    public function string(string $key): string;

    public function int(string $key): int;

    public function bool(string $key): bool;

    /**
     * @return array<array-key, mixed>
     */
    public function array(string $key): array;
}
```

## Configuration compilation

```text
config/*.php
.env values
secret providers
      |
      v
Validation
      |
      v
Immutable configuration tree
      |
      v
var/cache/config.php
```

Never place actual secrets into a committed cache artifact.

---

# 17. Service Providers and Modules

A provider should register services without running application logic.

```php
interface ServiceProvider
{
    public function register(
        ContainerBuilder $container
    ): void;
}
```

Optional boot phase:

```php
interface BootableServiceProvider extends ServiceProvider
{
    public function boot(
        Application $application
    ): void;
}
```

Keep the distinction clear:

```text
register():
    Define services.

boot():
    Perform operations requiring a completed container.
```

Avoid resolving services while registration is still happening. Doing so creates order-dependent bugs.

## Modular application support

A module can contribute:

```text
Services
Routes
Commands
Configuration schema
Event listeners
Migrations
Views
Translations
```

But all contributions should compile into standard framework registries.

---

# 18. Controller Resolution

Separate:

```text
Route match
Controller resolution
Argument resolution
Controller invocation
Result normalization
```

## Controller resolver

It converts:

```php
ShowUser::class
```

or:

```php
[UserController::class, 'show']
```

into a callable.

## Argument resolver

An ordered resolver chain can support:

```php
public function __invoke(
    ServerRequestInterface $request,
    UserId $id,
    CurrentUser $user,
    UserRepository $users,
): ResponseInterface {
}
```

Possible resolvers:

```text
1. RequestResolver
2. RouteParameterResolver
3. AttributeResolver
4. DTOResolver
5. AuthenticatedUserResolver
6. ContainerServiceResolver
7. DefaultValueResolver
```

Every resolver should say:

```php
public function supports(
    ReflectionParameter $parameter,
    InvocationContext $context,
): bool;
```

Do not allow two resolvers to silently compete. Detect ambiguity.

## Result normalizer

Controllers may return:

```text
ResponseInterface
ViewModel
JsonSerializable
array
string
null
```

But unrestricted controller return types create magic.

A safer initial design is:

```text
Core:
    ResponseInterface only

Optional full-stack package:
    ViewModel
    JsonResponse data
    RedirectResponse
```

---

# 19. Error and Exception Architecture

Separate errors into layers:

```text
Throwable
   |
   v
Exception Mapper
   |
   v
Error Representation
   |
   v
Response Factory
   |
   v
HTML / JSON / Problem Details Response
```

## Exception mapping

```php
interface ExceptionMapper
{
    public function supports(Throwable $exception): bool;

    public function map(
        Throwable $exception,
        ServerRequestInterface $request,
    ): ErrorRepresentation;
}
```

Examples:

```text
ValidationException       -> 422
AuthenticationException   -> 401
AuthorizationException    -> 403
ResourceNotFound          -> 404
MethodNotAllowed          -> 405
ConflictException         -> 409
Unexpected Throwable      -> 500
```

For APIs, support RFC 9457 Problem Details. RFC 9457 defines a machine-readable HTTP error format and replaced RFC 7807. ([RFC Editor][7])

Example:

```json
{
  "type": "https://docs.example.com/problems/validation",
  "title": "Request validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/problems/01JABC...",
  "errors": {
    "email": [
      "A valid email address is required."
    ]
  }
}
```

## Development versus production

Development response:

```text
Exception class
Message
File and line
Stack trace
Request details
Previous exception
```

Production response:

```text
Generic public message
Request/correlation ID
No stack trace
No credentials
No SQL
No filesystem path
```

Detailed information goes to the logger, not the client.

---

# 20. Event System

Use PSR-14 for synchronous events. PSR-14 standardizes dispatching and listening so components can integrate without depending on a particular framework implementation. ([PHP-FIG][8])

## Keep three concepts separate

### Framework lifecycle events

Examples:

```text
ApplicationBooted
RequestReceived
RouteMatched
ResponseCreated
ApplicationTerminating
```

### Domain events

Examples:

```text
OrderPlaced
PaymentCaptured
UserRegistered
```

### Asynchronous messages

Examples:

```text
SendWelcomeEmail
GenerateInvoice
SynchronizeCustomer
```

Do not treat these as the same system.

## Avoid event abuse

Use an event when:

* Zero or more listeners may react.
* The sender does not require a result.
* Extension without modifying the sender is useful.

Use a direct interface call when:

* The operation is required.
* The caller needs a result.
* Failure must affect control flow.
* Listener ordering would create hidden behavior.

Bad:

```php
$dispatcher->dispatch(new PleaseAuthorizePayment($order));
```

Better:

```php
$result = $paymentAuthorizer->authorize($order);
```

Then:

```php
$dispatcher->dispatch(
    new PaymentAuthorized($order->id(), $result->reference())
);
```

---

# 21. Logging and Observability

Use PSR-3 at the framework boundary. PSR-3 exists so libraries can write to centralized application logs without depending on a particular logging implementation. ([PHP-FIG][9])

## Structured context

```php
$logger->error(
    'Request processing failed.',
    [
        'request_id' => $requestId,
        'exception' => $exception,
        'route' => $routeName,
        'method' => $request->getMethod(),
        'path' => $request->getUri()->getPath(),
    ],
);
```

## Never log by default

* Passwords.
* Session IDs.
* Authorization headers.
* API keys.
* Complete payment information.
* Raw uploaded files.
* Full request bodies.
* Database connection credentials.

## Framework observability contracts

Design internal concepts for:

```text
Request ID
Correlation ID
Timer
Counter
Span
Context propagation
Exception reporting
```

Do not make a draft standard a permanent public dependency until it is accepted and stable.

---

# 22. Response Emission

A response emitter is a transport adapter.

It should:

1. Validate that headers have not already been sent.
2. Set the status code.
3. Emit headers.
4. Stream the body.
5. Avoid loading large files into memory.
6. Correctly handle HEAD requests.
7. Respect seekable and non-seekable streams.
8. Be separately testable.

```php
interface ResponseEmitter
{
    public function emit(ResponseInterface $response): void;
}
```

Do not put `header()` calls inside your response objects.

The response represents data. The emitter communicates with the PHP runtime.

---

# 23. Console Kernel

The CLI should use the same application configuration and container, but a separate kernel.

```text
bin/framework
      |
      v
Console Application
      |
      v
Command Registry
      |
      v
Input Parsing
      |
      v
Command Resolution
      |
      v
Execution
      |
      v
Exit Code
```

## Command contract

```php
interface Command
{
    public function name(): string;

    public function description(): string;

    public function execute(
        Input $input,
        Output $output,
    ): int;
}
```

Initial commands:

```text
framework about
framework routes:list
framework container:debug
framework config:show
framework cache:clear
framework make:controller
framework make:middleware
framework test
framework serve
```

Diagnostic commands are more important than code generators.

A framework becomes maintainable when developers can inspect what it built.

---

# 24. Developer Introspection

Provide tools for answering:

```text
Which route matched?
Why was this service selected?
Which provider registered this binding?
Which middleware ran?
Which configuration source set this value?
Which listeners receive this event?
Why did autowiring fail?
```

Example:

```text
$ bin/framework container:debug App\Service\CheckoutService

Service: App\Service\CheckoutService
Lifetime: singleton
Definition: autowired

Dependencies:
  App\Repository\OrderRepository
    -> App\Repository\PdoOrderRepository

  App\Payment\PaymentGateway
    -> App\Payment\StripePaymentGateway

  Psr\Log\LoggerInterface
    -> App\Logging\JsonLogger
```

This is how you make advanced machinery understandable.

---

# 25. Validation

Keep validation independent from HTTP.

Bad:

```php
$validator->validateRequest($request);
```

Better:

```php
$violations = $validator->validate(
    value: $registerUser,
    rules: $rules,
);
```

Then the HTTP adapter converts violations into a response.

## Components

```text
Rule
Violation
ViolationList
Validator
ValidationContext
Message Translator
Metadata Loader
```

Example:

```php
final readonly class RegisterUser
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
    ) {
    }
}
```

```php
$rules = [
    'name' => [
        new Required(),
        new Length(min: 2, max: 100),
    ],
    'email' => [
        new Required(),
        new Email(),
    ],
    'password' => [
        new Required(),
        new Length(min: 12),
    ],
];
```

Never make validation rules responsible for persistence, HTTP responses or translation rendering.

---

# 26. Database Roadmap

Do not build an ORM first.

An ORM is one of the largest and most difficult framework components because it combines:

* Metadata.
* Query generation.
* Identity maps.
* Change tracking.
* Relationships.
* Unit of work.
* Transactions.
* Lazy loading.
* Hydration.
* Caching.
* Schema mapping.
* Database differences.

Start with:

```text
1. Connection abstraction
2. Transaction manager
3. Parameterized query execution
4. Query result abstraction
5. Query builder
6. Migration system
7. Repository helpers
8. Optional data mapper
9. ORM only after substantial experience
```

## Connection interface

```php
interface Connection
{
    /**
     * @param array<string|int, mixed> $parameters
     */
    public function execute(
        string $sql,
        array $parameters = [],
    ): ExecutionResult;

    /**
     * @param array<string|int, mixed> $parameters
     */
    public function fetchAll(
        string $sql,
        array $parameters = [],
    ): array;

    public function transaction(callable $callback): mixed;
}
```

Use PDO prepared statements and bind user values instead of inserting user input directly into SQL strings. The PHP manual explicitly recommends parameter markers for user input. ([PHP][10])

## Transaction behavior

Test:

* Commit.
* Rollback.
* Exception rollback.
* Nested transaction policy.
* Savepoint support.
* Connection loss.
* Retry policy.
* Read/write connection routing.

## Query builder boundary

A query builder should generate:

```php
final readonly class CompiledQuery
{
    /**
     * @param array<string|int, mixed> $parameters
     */
    public function __construct(
        public string $sql,
        public array $parameters,
    ) {
    }
}
```

It should not directly execute the query.

---

# 27. Migration System

Migration concerns:

```text
Migration discovery
Version tracking
Ordering
Up execution
Down execution
Transactions
Locks
Dry run
SQL preview
Checksums
Environment protection
```

## Production safety

Require an explicit flag for destructive production operations:

```text
framework migrate --environment=production --force
```

Support:

```text
framework migrate
framework migrate:status
framework migrate:rollback
framework migrate:plan
framework migrate --dry-run
```

Never silently run destructive schema changes during ordinary HTTP boot.

---

# 28. Views and Templates

Do not build a template language during your first year.

Start with:

```text
ViewRenderer interface
Native PHP renderer
Template data object
Escaper
Layout support
```

```php
interface ViewRenderer
{
    /**
     * @param array<string, mixed> $data
     */
    public function render(
        string $template,
        array $data = [],
    ): string;
}
```

## Escaping must be contextual

Separate:

```text
HTML text escaping
HTML attribute escaping
JavaScript escaping
CSS escaping
URL escaping
```

A single universal `escape()` method invites misuse.

Provide adapters later for existing template systems rather than making your core dependent on one.

---

# 29. Sessions and Authentication

Separate:

```text
Session storage
Session middleware
Authentication
Authorization
CSRF
Password hashing
Token authentication
```

## Authentication contract

```php
interface Authenticator
{
    public function authenticate(
        ServerRequestInterface $request
    ): AuthenticationResult;
}
```

## Authorization contract

```php
interface AuthorizationChecker
{
    public function isGranted(
        mixed $permission,
        mixed $subject = null,
    ): bool;
}
```

Do not merge identity and authorization into one global helper.

## Password handling

Use PHP’s password APIs rather than inventing hashing schemes. `password_hash()` creates hashes using strong one-way algorithms, while `password_verify()` reads the required algorithm and salt information from the stored hash. ([PHP][11])

## Session security defaults

Design for:

* Secure cookies under HTTPS.
* HttpOnly cookies.
* SameSite policy.
* Session ID regeneration after authentication.
* Idle expiration.
* Absolute expiration.
* Logout invalidation.
* CSRF protection for cookie-authenticated state changes.
* Configurable storage adapters.

---

# 30. Queue and Message System

Do not couple message definitions to Redis, RabbitMQ or a particular service.

```php
interface MessageBus
{
    public function dispatch(object $message): DispatchResult;
}
```

Separate:

```text
Message serialization
Transport
Retry strategy
Failure storage
Middleware
Handler resolution
Worker
Acknowledgment
```

## Queue middleware examples

```text
Logging
Tracing
Transaction handling
Retry
Deduplication
Rate limiting
Handler invocation
```

## Required production concepts

* At-least-once delivery.
* Idempotent handlers.
* Retry delays.
* Maximum attempts.
* Dead-letter storage.
* Poison-message handling.
* Graceful worker shutdown.
* Memory reset between jobs.
* Visibility timeout.
* Unique message IDs.

Never promise “exactly once” behavior unless the entire storage and side-effect model truly guarantees it.

---

# 31. Scheduler

The scheduler should calculate what should run. A system cron or worker should trigger it.

```php
$schedule->command('reports:daily')
    ->dailyAt('02:00')
    ->withoutOverlapping()
    ->onOneServer();
```

Separate:

```text
Schedule definition
Due-event calculation
Locking
Execution
Logging
Failure reporting
```

Do not put an infinite scheduler loop in the core HTTP package.

---

# 32. Security Engineering

Treat security as a design track, not a final audit.

OWASP ASVS 5.0 provides a structured basis for testing application security controls and a requirements list for secure development. Use it as a framework security checklist. ([OWASP Foundation][12])

## Threat-model each package

For the router:

```text
Path confusion
Encoded separators
Host-header attacks
Route shadowing
Regular-expression denial of service
Unsafe redirects
```

For the container:

```text
Untrusted class names
Arbitrary object construction
Sensitive values in debug output
Unsafe deserialization
```

For views:

```text
Cross-site scripting
Path traversal
Template injection
Information disclosure
```

For uploads:

```text
Filename traversal
MIME confusion
Size exhaustion
Temporary-file misuse
Executable uploads
```

For database:

```text
SQL injection
Credential disclosure
Transaction confusion
Unsafe identifier interpolation
```

## Security files

Your repository should contain:

```text
SECURITY.md
Threat model
Supported versions
Private reporting instructions
Disclosure policy
Patch policy
```

GitHub repository security advisories allow maintainers to privately discuss, patch and publish vulnerabilities, and eligible projects can request CVE identifiers through the advisory process. ([GitHub Docs][13])

## Dependency security

Run:

```bash
composer validate --strict
composer audit
```

Composer’s audit command checks installed packages against security advisories and can also identify abandoned or malware-flagged packages under its dependency policy system. ([Composer][14])

---

# 33. Testing Strategy

A framework requires more than unit tests.

```text
                 End-to-End Tests
             Functional Application Tests
               Integration Tests
                 Contract Tests
                   Unit Tests
```

## Unit tests

Test isolated algorithms:

* Route parsing.
* Header normalization.
* Container graph resolution.
* Configuration merging.
* Query compilation.
* Validation rules.

## Contract tests

A contract test suite verifies every implementation of an interface.

Example:

```php
abstract class ContainerContractTest extends TestCase
{
    abstract protected function createContainer(): ContainerInterface;

    public function testItReturnsRegisteredService(): void
    {
        $container = $this->createContainer();

        self::assertInstanceOf(
            ExampleService::class,
            $container->get(ExampleService::class),
        );
    }
}
```

Then:

```php
final class RuntimeContainerTest
    extends ContainerContractTest
{
    protected function createContainer(): ContainerInterface
    {
        return RuntimeContainerFactory::createForTesting();
    }
}
```

And:

```php
final class CompiledContainerTest
    extends ContainerContractTest
{
    protected function createContainer(): ContainerInterface
    {
        return CompiledContainerFactory::createForTesting();
    }
}
```

Both implementations must satisfy identical behavior.

## Integration tests

Test boundaries:

```text
Router + middleware
Container + providers
Database + transactions
Session + cookies
Console + command registry
```

## Functional tests

Boot an actual example application and send requests through the complete kernel.

## Property-based testing

Useful properties:

```text
Generating and then matching a URL preserves parameters.

Adding an unrelated route does not change an existing static match.

Header lookup is case-insensitive.

Container compilation does not change service behavior.

Configuration cache output equals uncached configuration.
```

## Mutation testing

Mutation testing identifies tests that execute code without meaningfully checking its behavior.

Use it after ordinary test coverage is mature.

## Static analysis

PHPStan currently provides levels 0 through 10, with level 10 being the strictest. For greenfield framework code, target level 10 from the beginning rather than building a large baseline of ignored problems. ([PHPStan][15])

Recommended quality checks:

```bash
composer validate --strict
vendor/bin/phpunit
vendor/bin/phpstan analyse
vendor/bin/php-cs-fixer check
composer audit
```

---

# 34. CI Pipeline

A pull request should run:

```text
1. Composer validation
2. Coding-style validation
3. Static analysis
4. Unit tests
5. Contract tests
6. Integration tests
7. Lowest dependency versions
8. Highest dependency versions
9. PHP 8.4
10. PHP 8.5
11. Dependency security audit
12. Documentation link/code validation
```

## Dependency matrix

Test both:

```text
composer update --prefer-lowest
composer update
```

The lowest-dependency run finds overly broad or incorrect Composer constraints.

## Protected main branch

Require:

* Pull requests.
* Passing CI.
* At least one review.
* No direct pushes.
* Resolved conversations.
* Optional signed commits.
* Optional linear history.

GitHub protected branches can require successful status checks before changes are merged. ([GitHub Docs][16])

---

# 35. Performance Engineering

Do not begin by chasing benchmark headlines.

First define performance budgets.

Example:

```text
Warm framework boot:        < 5 ms
Static route match:         < 10 µs
Container service lookup:   < 2 µs
Framework memory baseline:  < 8 MB
No reflection after cached production boot
No filesystem scans per production request
```

These are project goals, not universal promises.

## Measure separately

```text
Cold boot
Warm boot
Container compilation
Container lookup
Static route matching
Dynamic route matching
Middleware overhead
JSON response creation
Template rendering
Database adapter overhead
Memory retention across 10,000 worker requests
```

## Optimization order

1. Measure.
2. Profile.
3. Identify meaningful bottlenecks.
4. Improve algorithms.
5. Cache stable metadata.
6. Reduce allocations where justified.
7. Repeat measurements.
8. Document tradeoffs.

## Production compilation targets

Compile:

```text
Configuration
Container definitions
Route tables
Event listener maps
Attribute metadata
Command registry
Validation metadata
```

Never require an application to scan its entire source tree on every production request.

---

# 36. Long-Running Runtime Safety

Your framework should define reset behavior.

```php
interface Resettable
{
    public function reset(): void;
}
```

After every request or queue job:

```text
Clear request context
Clear authenticated identity
Close or validate connections
Clear request-level caches
Clear session state
Reset collectors
Remove temporary files
Flush buffered logs
```

## Tests

Run thousands of synthetic requests through the same application object:

```php
for ($i = 0; $i < 10_000; $i++) {
    $response = $kernel->handle(
        RequestFactory::forUser("user-{$i}")
    );

    self::assertSame(
        "user-{$i}",
        (string) $response->getBody(),
    );

    $resetter->reset();
}
```

Check:

* Memory growth.
* Cross-request identity leakage.
* Middleware state.
* Container scopes.
* Database transactions.
* Event listener accumulation.

---

# 37. Public API and Backward Compatibility

Your public API includes more than public PHP methods.

It can include:

* Interfaces.
* Public classes.
* Constructor signatures.
* Exceptions.
* Configuration keys.
* Route syntax.
* Attribute arguments.
* CLI names and flags.
* Event classes.
* Service IDs.
* Generated file formats.
* Database migration metadata.
* Serialized message formats.
* Documented behaviors.

Mark internals clearly:

```php
/**
 * @internal
 */
final class CompiledDefinitionGraph
{
}
```

## Interface warning

An interface is difficult to change after 1.0.

Adding a new method to a public interface is usually breaking for third-party implementers.

Prefer small, capability-specific interfaces:

```php
interface CacheReader
{
    public function get(string $key): mixed;
}

interface CacheWriter
{
    public function put(
        string $key,
        mixed $value,
        ?DateInterval $ttl = null,
    ): void;
}
```

Instead of one enormous interface.

---

# 38. Release Strategy

Use Semantic Versioning.

Semantic Versioning defines:

* Major releases for incompatible public API changes.
* Minor releases for backward-compatible features and deprecations.
* Patch releases for backward-compatible fixes.
* `0.y.z` for initial unstable development.
* `1.0.0` as the declaration of a stable public API. ([Semantic Versioning][17])

## Recommended release path

```text
0.1.0  HTTP messages and basic kernel
0.2.0  Middleware pipeline
0.3.0  Router
0.4.0  Container and providers
0.5.0  Configuration and cache compilation
0.6.0  Console
0.7.0  Error handling and observability
0.8.0  Security/session/validation packages
0.9.0  API freeze candidate
1.0.0  Stable contracts and support policy
```

## Deprecation policy

For stable versions:

```text
1. Introduce replacement.
2. Mark old API deprecated.
3. Document migration.
4. Emit development-only deprecation notice where appropriate.
5. Keep it for at least one minor release.
6. Remove only in the next major release.
```

Semantic Versioning recommends releasing deprecations in a minor version and providing users time to migrate before removal in a major release. ([Semantic Versioning][17])

---

# 39. Framework RFC Process

Before 1.0, you may make decisions through maintainers and ADRs.

As the community grows, adopt an RFC process.

PHP core’s own RFC workflow includes proposal preparation, public discussion, recording positive and negative arguments, implementation considerations, voting and documenting the final outcome. ([PHP Wiki][18])

Your RFC template:

```text
Title
Author
Status
Created date
Target release

Summary
Motivation
Goals
Non-goals
Proposed API
Detailed behavior
Error behavior
Security considerations
Performance considerations
Backward compatibility
Alternatives considered
Migration plan
Reference implementation
Open questions
Decision
```

## RFC lifecycle

```text
Draft
  |
  v
Discussion
  |
  v
Prototype
  |
  v
Final Comment Period
  |
  +------> Rejected
  |
  v
Accepted
  |
  v
Implemented
```

Do not use voting as a substitute for technical reasoning.

---

# 40. Documentation Architecture

Documentation is one of the framework’s primary products.

Organize it into four categories:

## Tutorials

Learning-oriented:

```text
Build your first application
Build a JSON API
Add database access
Create middleware
Create a console command
```

## How-to guides

Task-oriented:

```text
Configure trusted proxies
Replace the router
Create a custom exception mapper
Use a different PSR-7 implementation
Deploy with PHP-FPM
Run in a persistent worker
```

## Explanation

Understanding-oriented:

```text
Request lifecycle
Container scopes
Why controllers return responses
How route compilation works
Framework events versus domain events
```

## Reference

Exact technical information:

```text
Configuration keys
CLI commands
Attributes
Interfaces
Exceptions
Middleware order
Service IDs
```

## Executable documentation

Code samples should be tested.

Place examples in files that CI runs rather than manually copying unverified snippets into Markdown.

---

# 41. Open-Source Project Structure

Before public release, add:

```text
README.md
LICENSE
CONTRIBUTING.md
CODE_OF_CONDUCT.md
SECURITY.md
SUPPORT.md
GOVERNANCE.md
CHANGELOG.md
UPGRADE.md
```

## README structure

```text
1. One-sentence purpose
2. Status warning
3. Requirements
4. Installation
5. Smallest working example
6. Philosophy
7. Documentation
8. Security reporting
9. Contributing
10. License
```

## Issue templates

Create:

```text
Bug report
Feature proposal
Documentation problem
Performance regression
Security redirect
```

Security reports must not be filed as public issues.

---

# 42. Governance Model

A possible evolution:

## Stage 1: Founder-led

You make final decisions but document them.

## Stage 2: Maintainer team

Maintainers own packages:

```text
HTTP maintainer
Container maintainer
Router maintainer
Database maintainer
Documentation maintainer
Security maintainer
Release manager
```

## Stage 3: Technical steering group

Responsibilities:

* API governance.
* Release approval.
* Security response.
* Maintainer appointments.
* RFC decisions.
* Conflict resolution.

## Bus-factor protection

Require:

* At least two release-capable maintainers.
* Documented release procedure.
* Organization-owned package publishing.
* Organization-controlled domains.
* Multiple security contacts.
* Two-factor authentication.
* Backup ownership of repositories and package namespaces.

---

# 43. Common Framework-Creator Mistakes

## 1. Building an ORM first

It consumes the project before the kernel is stable.

## 2. Copying Laravel syntax

You inherit comparisons without offering a new architectural reason to exist.

## 3. Making everything static

Static APIs are convenient initially but complicate replacement, isolation and long-running workers.

## 4. Making an interface for every class

Interfaces are valuable at substitution boundaries, not as decoration.

## 5. Publishing too many packages early

You create release complexity before understanding your boundaries.

## 6. Hiding behavior in traits

Traits can make dependencies and state difficult to understand.

## 7. Excessive lifecycle events

The execution flow becomes invisible.

## 8. Using the container everywhere

Application code becomes a service-location graph rather than an object model.

## 9. Promising 1.0 too early

You become trapped by accidental APIs.

## 10. Supporting too many PHP versions

Compatibility code increases maintenance and prevents clean designs.

## 11. Optimizing benchmark demos

A router benchmark does not measure real application performance.

## 12. Ignoring error messages

Developer-facing failures are part of the framework’s UX.

## 13. No dogfooding application

A framework that is not used to build real applications will develop unrealistic APIs.

---

# 44. Realistic 18-Month Roadmap

## Months 1–2: Architecture and foundations

Deliver:

* Vision.
* Non-goals.
* Coding standard.
* CI.
* PHPStan level 10.
* ADR process.
* Package boundaries.
* Minimal Composer package.
* Basic HTTP experiments.

## Months 3–4: HTTP runtime

Deliver:

* PSR-7 implementation.
* PSR-17 factories.
* Request creation.
* Response emitter.
* PSR-15 pipeline.
* Error middleware.
* Functional test harness.

Release:

```text
0.1.0
```

## Months 5–6: Router and controller system

Deliver:

* Route definitions.
* Static and dynamic matching.
* URL generation.
* Route groups.
* Controller resolution.
* Argument resolution.
* Route caching.

Release:

```text
0.2.0
```

## Months 7–8: Container and bootstrap

Deliver:

* PSR-11 container.
* Definitions.
* Autowiring.
* Cycle detection.
* Scopes.
* Providers.
* Container compilation.
* Configuration system.

Release:

```text
0.3.0
```

## Months 9–10: Console and diagnostics

Deliver:

* Console kernel.
* Command registry.
* Route inspection.
* Container inspection.
* Configuration inspection.
* Cache commands.
* Development server adapter.

Release:

```text
0.4.0
```

## Months 11–12: Application components

Deliver:

* Validation.
* Session.
* CSRF.
* Authentication contracts.
* Authorization.
* Native view renderer.
* Cache adapters.
* Translation contracts.

Release:

```text
0.5.0
```

## Months 13–14: Data and background processing

Deliver:

* PDO connection layer.
* Query builder.
* Transactions.
* Migrations.
* Message bus.
* In-memory queue transport.
* Worker lifecycle.

Release:

```text
0.6.0
```

## Months 15–16: Dogfooding and stabilization

Build at least:

```text
One REST API
One server-rendered application
One CLI application
One worker application
```

Record every point where the framework feels confusing.

Release:

```text
0.7.0 and 0.8.0
```

## Months 17–18: API freeze

Deliver:

* Complete reference documentation.
* Security review.
* Performance baseline.
* Upgrade policy.
* Deprecation mechanism.
* Package split if justified.
* Third-party integration examples.
* Public RFC review.

Release:

```text
0.9.0
1.0.0-rc.1
```

Do not release 1.0 until external developers can build applications without your direct help.

---

# 45. Your First 30 Days

## Week 1: Write the constitution

Create:

```text
vision.md
non-goals.md
principles.md
package-boundaries.md
support-policy.md
```

Decide:

```text
Minimum PHP 8.4
MIT or BSD-3-Clause license
PSR-7/15 outer kernel
PSR-11 container boundary
PSR-14 events
No ORM in core
No static facades in core
```

## Week 2: Establish engineering quality

Configure:

```text
Composer
PHPUnit
PHPStan level 10
Coding-style checker
GitHub Actions
Composer audit
Branch protection
```

Write one intentionally tiny package with perfect tests and documentation.

## Week 3: HTTP messages

Implement:

```text
Stream
Uri
Request
ServerRequest
Response
UploadedFile
PSR-17 factories
```

Write contract tests for every immutable operation.

## Week 4: First request lifecycle

Implement:

```text
Request from globals
Middleware pipeline
Fallback handler
Response emitter
Error middleware
Hello-world application
```

Success condition:

```bash
composer create-project yourframework/skeleton example
cd example
php -S localhost:8000 -t public
```

And:

```text
GET /hello/Qasim
```

returns:

```json
{
  "message": "Hello, Qasim"
}
```

---

# 46. Minimum Viable Framework

Your first usable version needs only:

```text
Composer package
Application bootstrap
PSR-7 messages
PSR-15 middleware
PSR-17 factories
PSR-11 container
Router
Controller dispatcher
Error handling
Response emitter
Configuration
Console command runner
Logger integration
Tests
Documentation
```

It does not need:

```text
ORM
Blade-style compiler
Queue dashboard
Cloud integrations
Realtime broadcasting
Admin panel
Payment package
GraphQL package
AI package
```

A reliable small framework is more valuable than an incomplete giant framework.

---

# 47. Definition of “World-Class”

Your framework is approaching world-class quality when:

### Architecture

* Package dependencies form a clean directed graph.
* The request lifecycle can be explained on one page.
* Components can be replaced independently.
* Persistent workers do not leak request state.

### API design

* Public APIs are documented.
* Errors are descriptive.
* Extension points are intentional.
* Internal APIs are clearly marked.
* Deprecations have migration paths.

### Quality

* CI tests all supported PHP versions.
* Static analysis runs at maximum strictness.
* Contract tests protect replaceable components.
* Security checks run automatically.
* Performance changes are measured.

### Developer experience

* A project can be created in minutes.
* Routes and services can be inspected.
* Documentation is searchable and versioned.
* Debug output explains causes and possible fixes.

### Open source

* Contributions have a documented path.
* Security reports have a private path.
* Releases follow a predictable policy.
* Governance does not depend permanently on one person.

### Adoption

* Developers build real applications with it.
* Third parties publish integrations.
* Applications can upgrade without rewrites.
* Users trust the maintainers’ compatibility promises.

The correct first milestone is not “compete with Laravel.” It is:

> Create the smallest framework kernel whose behavior is explicit, thoroughly tested, standards-compatible and pleasant enough that you personally choose it for a real production application.

[1]: https://www.php.net/supported-versions.php?utm_source=chatgpt.com "PHP: Supported Versions"
[2]: https://www.php-fig.org/per/coding-style/?utm_source=chatgpt.com "PER Coding Style 3.0"
[3]: https://www.php-fig.org/psr/?utm_source=chatgpt.com "PHP Standards Recommendations"
[4]: https://symfony.com/doc/current/components/http_kernel.html?utm_source=chatgpt.com "The HttpKernel Component (Symfony Docs)"
[5]: https://getcomposer.org/doc/01-basic-usage.md?utm_source=chatgpt.com "Basic usage"
[6]: https://www.php-fig.org/psr/psr-7/?utm_source=chatgpt.com "PSR-7: HTTP message interfaces"
[7]: https://www.rfc-editor.org/rfc/rfc9457.html "RFC 9457: Problem Details for HTTP APIs"
[8]: https://www.php-fig.org/psr/psr-14/?utm_source=chatgpt.com "PSR-14: Event Dispatcher"
[9]: https://www.php-fig.org/psr/psr-3/?utm_source=chatgpt.com "PSR-3: Logger Interface"
[10]: https://www.php.net/manual/en/pdo.prepare.php?utm_source=chatgpt.com "PDO::prepare - Manual"
[11]: https://www.php.net/manual/en/function.password-hash.php?utm_source=chatgpt.com "password_hash - Manual"
[12]: https://owasp.org/www-project-application-security-verification-standard/ "OWASP Application Security Verification Standard (ASVS) | OWASP Foundation"
[13]: https://docs.github.com/en/code-security/security-advisories/working-with-repository-security-advisories/about-repository-security-advisories "Repository security advisories - GitHub Docs"
[14]: https://getcomposer.org/doc/03-cli.md?utm_source=chatgpt.com "Command-line interface / Commands"
[15]: https://phpstan.org/user-guide/rule-levels?utm_source=chatgpt.com "Rule Levels"
[16]: https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches?utm_source=chatgpt.com "About protected branches"
[17]: https://semver.org/ "Semantic Versioning 2.0.0 | Semantic Versioning"
[18]: https://wiki.php.net/rfc/howto "PHP: rfc:howto"

.......................................................................................................................
GameHull mein live chat integrate karne ke liye aapko existing contact form ko **two-way support conversation system** mein convert karna hoga.

Main aapke existing **PHP/MySQL admin architecture** ke according yeh structure recommend karta hoon.

## Required Workflow

```text
User starts chat
→ Chat goes to Unassigned Queue
→ Support Agent A claims it
→ Only Agent A can reply
→ Agent A resolves or transfers it
→ After transfer, only Agent B can reply
```

Sirf frontend button disable karna enough nahi hai. Backend aur database level par bhi agent ownership enforce karni hogi.

---

# 1. Database Tables

Minimum three new tables banayein.

## `support_conversations`

Har support chat ka main record:

```text
id
user_id
subject
category
priority
status
assigned_agent_id
claimed_at
last_message_at
resolved_at
closed_at
created_at
updated_at
```

Recommended statuses:

```text
open
claimed
waiting_for_user
waiting_for_support
resolved
closed
```

Meaning:

* `open`: kisi agent ne claim nahi ki
* `claimed`: agent assigned hai
* `waiting_for_user`: support ne reply kar diya
* `waiting_for_support`: user ne reply kiya
* `resolved`: issue solve ho gaya
* `closed`: conversation permanently close

---

## `support_messages`

Conversation ke tamam user aur admin messages:

```text
id
conversation_id
sender_type
sender_user_id
sender_agent_id
message
attachment_path
is_internal_note
read_at
created_at
```

`sender_type` values:

```text
user
agent
system
```

Internal notes sirf admin/support ko show hon:

```text
is_internal_note = 1
```

User ko internal notes kabhi expose na karein.

---

## `support_chat_transfers`

Agent-transfer history:

```text
id
conversation_id
from_agent_id
to_agent_id
transferred_by
transfer_reason
created_at
```

Is se audit trail rahega ke chat kis agent se kis agent ko transfer hui.

---

# 2. User Chat Interface

User side par in mein se koi interface create karein:

```text
/support
```

ya floating chat button:

```text
Help & Support
```

User interface mein:

* Start New Conversation
* Subject
* Category
* Message
* Attachment
* Conversation history
* Agent reply
* Unread message count
* Close conversation

Categories example:

```text
Game Account Issue
Deposit Issue
Withdrawal Issue
Affiliate Issue
Password Reset
Technical Problem
Other
```

User sirf apni conversation access kar sake:

```sql
WHERE support_conversations.user_id = logged_in_user_id
```

---

# 3. Admin Support Panel

Admin route:

```text
/admin/support/chats
```

Recommended tabs:

```text
Unassigned
My Chats
Waiting for Support
Waiting for User
Resolved
Closed
```

Each chat row mein show karein:

```text
Chat ID
User
Subject
Category
Priority
Status
Assigned Agent
Last Message
Last Activity
```

Admin actions:

```text
Claim Chat
Reply
Add Internal Note
Transfer Chat
Resolve Chat
Close Chat
Reopen Chat
```

---

# 4. Claim Chat Logic

Yeh sabse important part hai.

Jab Agent A **Claim Chat** click kare, backend ko atomic update karni chahiye:

```sql
UPDATE support_conversations
SET
    assigned_agent_id = :agent_id,
    status = 'claimed',
    claimed_at = NOW()
WHERE id = :conversation_id
  AND assigned_agent_id IS NULL
  AND status = 'open';
```

Phir affected rows check karein:

```text
1 row updated → Agent A successfully claimed
0 rows updated → Another agent already claimed it
```

Agent B ko message show karein:

```text
This chat has already been claimed by another support agent.
```

Is atomic update ki wajah se do agents same time par chat claim nahi kar sakenge.

---

# 5. Agent Reply Permission

Admin reply endpoint par check karein:

```text
Is logged-in admin the assigned agent?
```

Backend rule:

```php
$conversation->assigned_agent_id === $loggedInAgentId
```

Agar match nahi karta:

```text
403 Forbidden
This conversation is assigned to another support agent.
```

Super Admin ke liye aap optionally override permission de sakte hain, lekin normal support agents sirf assigned chats mein reply karein.

---

# 6. Chat Transfer Logic

Route:

```text
POST /admin/support/chats/{id}/transfer
```

Required fields:

```text
to_agent_id
transfer_reason
```

Backend process:

1. Database transaction start karein.
2. Conversation row lock karein.
3. Confirm logged-in agent current assigned agent hai.
4. New agent active support member hai.
5. Transfer-history record create karein.
6. `assigned_agent_id` new agent se update karein.
7. System message add karein.
8. Transaction commit karein.

Example system message:

```text
Chat transferred from Agent A to Agent B.
```

Transfer complete hone ke baad Agent A reply nahi kar sakega. Sirf Agent B reply karega.

---

# 7. Required API Routes

## User routes

```text
POST /support/conversations
GET  /support/conversations
GET  /support/conversations/{id}
POST /support/conversations/{id}/messages
POST /support/conversations/{id}/close
```

## Admin routes

```text
GET  /admin/support/chats
GET  /admin/support/chats/{id}
POST /admin/support/chats/{id}/claim
POST /admin/support/chats/{id}/messages
POST /admin/support/chats/{id}/transfer
POST /admin/support/chats/{id}/resolve
POST /admin/support/chats/{id}/close
POST /admin/support/chats/{id}/reopen
```

## Notification route

```text
GET /support/unread-count
```

---

# 8. Real-Time Messages

Aapke paas do implementation options hain.

## Option A — AJAX Polling

Frontend har 3–5 seconds baad new messages check kare:

```text
GET /support/conversations/{id}/messages?after_id=123
```

Yeh existing PHP/MySQL project aur normal cPanel environment ke liye simplest option hai.

Advantages:

* Easy implementation
* Separate WebSocket server nahi chahiye
* Existing PHP hosting par kaam karega

Disadvantage:

* True instant chat nahi
* Repeated server requests hongi

## Option B — WebSockets

Real-time production chat ke liye private WebSocket channel use karein:

```text
private-support-chat.{conversationId}
```

Laravel currently Reverb, Pusher Channels aur Ably broadcasting drivers support karta hai. Private channels server-side authorization require karte hain, jo ensure karta hai ke sirf allowed user ya assigned support agent conversation subscribe kar sake. ([Laravel][1])

Recommended authorization:

```text
Allow when:
logged-in user owns conversation

OR

logged-in admin is assigned agent

OR

logged-in admin has super-admin permission
```

### GameHull ke liye recommendation

Agar GameHull normal PHP/custom framework aur cPanel par chal raha hai:

```text
Phase 1: AJAX polling
Phase 2: WebSocket real-time upgrade
```

Agar Laravel aur VPS/server access available hai, Laravel Reverb ya private Pusher/Ably channels use kiye ja sakte hain. Realtime presence functionality agent online/offline state show karne ke liye bhi use ho sakti hai. ([Ably Realtime][2])

---

# 9. Existing Contact Form Ka Kya Karna Hai?

Current contact form ko remove karna zaroori nahi.

Contact form submission ke baad:

```text
Create support_conversation
Create first support_message
Show conversation inside user support page
Show it inside admin Unassigned queue
```

Ab admin sirf message dekhega nahi, balki chat claim karke reply bhi kar sakega.

User ko notification mile:

```text
Support has replied to your conversation.
```

---

# 10. Notifications

User notifications:

* Support agent replied
* Chat transferred
* Issue resolved
* Chat closed

Agent notifications:

* New unassigned chat
* Chat transferred to agent
* User replied
* High-priority issue created

Notification channels:

```text
In-app notification
Unread badge
Email notification
```

Email mein sensitive chat content bhejne ke bajaye yeh message bhejna better hai:

```text
You have a new support response. Log in to view it.
```

---

# 11. Security Requirements

Har endpoint par:

```text
Authentication
CSRF protection
Conversation ownership
Agent assignment check
Input validation
Output escaping
Rate limiting
```

Important rules:

* User doosre user ki chat access na kar sake.
* Agent unassigned ya another-agent chat mein reply na kar sake.
* Chat ID modify karke authorization bypass na ho.
* Messages HTML-escaped hon.
* Passwords aur provider credentials chat mein automatically expose na hon.
* Attachments ka MIME type aur size validate ho.
* Executable files upload allow na hon.
* Every claim, transfer, resolve aur close action audit log mein ho.

---

# 12. UI States

## User side

```text
Support is reviewing your issue.
Waiting for your response.
Issue resolved.
Conversation closed.
```

## Admin side

```text
Unassigned
Claimed by You
Claimed by Agent Ahmed
Waiting for User
Waiting for Support
Resolved
```

Jab another agent claimed conversation open kare:

```text
Assigned to Agent Ahmed
Read-only
```

Reply box disabled hona chahiye.

---

# 13. Required QA Tests

Integration ke baad yeh tests karein:

1. User conversation create kar sakta hai.
2. User sirf apni conversations dekh sakta hai.
3. Two agents same time claim karein to sirf one succeeds.
4. Unassigned agent reply nahi kar sakta.
5. Assigned agent reply kar sakta hai.
6. Agent chat successfully transfer kar sakta hai.
7. Old agent transfer ke baad reply nahi kar sakta.
8. New agent transfer ke baad reply kar sakta hai.
9. User reply assigned agent ko show hota hai.
10. Resolve aur close actions properly work karte hain.
11. Closed conversation mein messages block hote hain.
12. Unauthorized chat ID request `403` ya `404` return karti hai.
13. Internal notes user ko show nahi hoti.
14. Attachments secure hain.
15. Unread counts correctly update hote hain.

---

# Recommended Integration Order

```text
1. Database migrations
2. Conversation and message models
3. User conversation APIs
4. Admin unassigned queue
5. Atomic Claim Chat logic
6. Agent reply permissions
7. Transfer functionality
8. User chat interface
9. Admin chat interface
10. AJAX polling or WebSockets
11. Notifications
12. Audit logs
13. Security and QA testing
```

Sabse pehle **conversation, message, claim aur transfer workflow** implement karein. Real-time WebSocket baad mein add kiya ja sakta hai; claim ownership database level par start se implement honi chahiye.

[1]: https://laravel.com/docs/12.x/broadcasting?utm_source=chatgpt.com "Broadcasting | Laravel 12.x - The clean stack for Artisans and agents"
[2]: https://ably.com/docs/presence-occupancy?utm_source=chatgpt.com "Ably Pub/Sub | Presence and occupancy overview"
