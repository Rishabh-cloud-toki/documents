# 01 — Object-level design patterns

Creational, structural and behavioural patterns (the Gang of Four set), plus the handful of
extra patterns every Java developer meets, plus concurrency patterns. This is the smallest
scale: **a few classes working together inside one component**.

Part of the [design patterns guide](README.md). Next: [application internals](02-application-internals.md).

> **Status: preview — not yet read.**

**Running example (used in every file):** a flight-booking system. At this scale the classes
are `Booking`, `Fare`, `Payment`, `Notification`.

**Naming note:** several pattern names recur at larger scales (Adapter, Facade, Bridge, Proxy,
Command, Mediator). Here they are marked *(object)* so they are not confused with their
integration or distributed namesakes.

## Contents

1. [Level 1 — the simple version](#level-1--the-simple-version)
2. [Part A — Creational patterns](#part-a--creational-patterns)
3. [Part B — Structural patterns](#part-b--structural-patterns)
4. [Part C — Behavioural patterns](#part-c--behavioural-patterns)
5. [Part D — Concurrency patterns](#part-d--concurrency-patterns)
6. [Level 3 — deep dive](#level-3--deep-dive)
7. [Check yourself](#check-yourself)

---

## Level 1 — the simple version

A **design pattern** is a named, proven solution to a problem that keeps recurring. It is not
code you copy; it is a *shape* you recognise. The name matters as much as the shape: saying
"use a Decorator" transfers a whole design in two words.

Every GoF pattern answers one of three questions:

| Family | Question it answers | One-line summary |
|---|---|---|
| **Creational** | *How do I make objects without hard-wiring which class?* | Hide `new`. |
| **Structural** | *How do I assemble objects into bigger things?* | Wrap, compose, adapt. |
| **Behavioural** | *How do objects share work and talk to each other?* | Move decisions out of `if/else` into objects. |

Three habits sit underneath almost all of them (see [SOLID](../architecture/solid-principles.md)):

- **Program to an interface**, not an implementation.
- **Favour composition over inheritance** — hold a collaborator, don't extend a parent.
- **Encapsulate what varies** — find the part that changes and put it behind an interface.

### Quick chooser — "I need to…"

| I need to… | Reach for |
|---|---|
| Create objects without the caller naming the concrete class | Factory Method / Abstract Factory |
| Build an object with many optional fields, readably | Builder |
| Guarantee one instance | Singleton (usually via DI scope instead) |
| Swap an algorithm at runtime | Strategy |
| Change behaviour when internal state changes | State |
| Notify many parties when something happens | Observer |
| Add behaviour to an object without subclassing | Decorator |
| Make an incompatible class fit my interface | Adapter |
| Give a messy subsystem one simple front door | Facade |
| Control access / lazy-load / cache a heavy object | Proxy |
| Treat a tree of things and a single thing the same way | Composite |
| Turn a request into an object (queue, undo, log it) | Command |
| Pass a request along a chain of possible handlers | Chain of Responsibility |
| Define the skeleton of an algorithm; let subclasses fill steps | Template Method |
| Add operations to a stable class hierarchy without editing it | Visitor |
| Capture and restore an object's state | Memento |
| Stop objects from referring to each other directly | Mediator |
| Avoid `null` checks | Null Object |

**Check yourself (L1):** (1) Which family would you search to stop writing `new StripeGateway()` everywhere?
(2) Why is the *name* part of the value of a pattern?

---

## Part A — Creational patterns

### Singleton
**Intent:** exactly one instance, with a global access point.
**Flight example:** a `FlightCatalogCache` that must be shared.

```java
public enum FlightCatalogCache {          // enum singleton: thread-safe, serialization-safe
    INSTANCE;
    private final Map<String, Flight> cache = new ConcurrentHashMap<>();
    public Flight get(String id) { return cache.get(id); }
}
```

- **Variants:** eager, lazy holder idiom, double-checked locking with `volatile`, enum.
- **Use when:** truly one of something exists (hardware handle, process-wide registry).
- **Avoid when:** you just want "one shared object" — it is global state, hard to test, hides dependencies, and breaks under multiple class loaders / clustered deployments.
- **Modern replacement:** let the container manage it — a Spring `@Component` is singleton-*scoped* and still injectable and mockable.
- **Watch:** "singleton" in Spring means one per `ApplicationContext`, not one per JVM.

### Factory Method
**Intent:** a method decides *which subclass* to instantiate; the caller works against the interface.

```java
interface PaymentGateway { Receipt charge(Money m); }

class GatewayFactory {
    static PaymentGateway forMethod(PaymentMethod m) {
        return switch (m) {
            case CARD   -> new StripeGateway();
            case UPI    -> new UpiGateway();
            case WALLET -> new WalletGateway();
        };
    }
}
```

- **Use when:** the concrete type depends on input/config and the caller shouldn't care.
- **Classic form:** abstract creator with a `createGateway()` method that subclasses override. The "simple factory" above is the form you will actually write most.
- **Trade-off:** one more indirection; the `switch` is a modification point (use a registry `Map<PaymentMethod, Supplier<…>>` to make it open for extension).
- **In the wild:** `Calendar.getInstance()`, `List.of()`, `NumberFormat.getInstance()`, Spring `FactoryBean`.

### Abstract Factory
**Intent:** create *families* of related objects that must be used together.
**Flight example:** per-region stack — `EuStack` makes `EuTaxCalculator + SepaPayment + GdprConsentForm`; `UsStack` makes the US equivalents. The caller can never mix an EU tax rule with a US payment rail.

- **Use when:** a consistent set of products varies together (regions, UI themes, DB vendors).
- **Avoid when:** only one product varies — plain Factory Method is enough.
- **Trade-off:** adding a *new product type* means changing every factory.
- **In the wild:** `javax.xml.parsers.DocumentBuilderFactory`, JDBC (`Connection` → `Statement`, `ResultSet` of one vendor).

### Builder
**Intent:** construct a complex object step by step; separate construction from representation.

```java
Booking b = Booking.builder()
        .passenger(p)
        .flight(f)
        .seat("14C")
        .meal(Meal.VEGAN)           // optional
        .insurance(false)           // optional
        .build();                   // validates invariants here
```

- **Use when:** many optional parameters ("telescoping constructor" problem), immutable objects, or construction needs validation as a whole.
- **Variants:** fluent builder (above); *Director* that scripts common recipes; step builders that enforce order at compile time.
- **Trade-off:** extra class (Lombok `@Builder` removes the boilerplate). Don't use for 2–3 required fields — a constructor or record is clearer.
- **In the wild:** `StringBuilder`, `HttpRequest.newBuilder()`, `Stream.Builder`, Lombok.

### Prototype
**Intent:** make new objects by *copying an existing one*.
**Flight example:** clone a configured `Itinerary` template for a "same trip next week" action.

- **Use when:** construction is costly, or you want a copy of a configured object without knowing its class.
- **Gotcha:** shallow vs deep copy. Java's `Cloneable`/`clone()` is widely considered broken — prefer a copy constructor, a `copy()`/`with…()` method, or records with "wither" methods.
- **In the wild:** `Object.clone`, JS `Object.create`, copy-on-write collections.

### Also worth knowing (creational extras)

| Pattern | One line | Note |
|---|---|---|
| **Object Pool** | Reuse expensive objects instead of creating them. | DB connection pools (HikariCP), thread pools. Must reset state on return. |
| **Dependency Injection** | Objects receive collaborators instead of creating them. | The modern answer to most creational problems; see [application internals](02-application-internals.md#dependency-injection--inversion-of-control). |
| **Lazy Initialization / Holder idiom** | Create on first use. | `static class Holder { static final X I = new X(); }` is lazy and thread-safe without locks. |
| **Static factory methods** | Named constructors: `Money.of(…)`, `Optional.of(…)`. | *Effective Java* item 1 — often better than a constructor. |

---

## Part B — Structural patterns

### Adapter (object)
**Intent:** convert one interface into another the client expects.
**Flight example:** your code expects `FlightSearch.search(Query)`; the airline's SOAP client has `getAvailability(XmlRequest)`.

```java
class AirlineSoapAdapter implements FlightSearch {
    private final AirlineSoapClient soap;
    public List<Flight> search(Query q) {
        return soap.getAvailability(toXml(q)).stream().map(this::toFlight).toList();
    }
}
```

- **Use when:** integrating legacy code or third-party libraries you can't change.
- **Forms:** object adapter (holds the adaptee — preferred) vs class adapter (inherits).
- **Scales up:** the same idea at system scale is the [Anti-Corruption Layer](06-integration-patterns.md#anti-corruption-layer) and the [Channel Adapter](06-integration-patterns.md#channel-adapter).
- **In the wild:** `InputStreamReader` (bytes → chars), `Arrays.asList`, Spring `HandlerAdapter`.

### Bridge (object)
**Intent:** split an abstraction from its implementation so each varies independently (two inheritance hierarchies become two hierarchies connected by composition).
**Flight example:** `Notification` (abstraction: `Booking confirmation`, `Delay alert`) × `Channel` (implementation: `Email`, `SMS`, `Push`). Without Bridge you'd have `EmailDelayAlert`, `SmsDelayAlert`, … (M × N classes); with it, M + N.

- **Use when:** you see a class explosion from two independent dimensions of variation.
- **vs Adapter:** Adapter fixes a mismatch *after the fact*; Bridge is designed up front.
- **In the wild:** JDBC (`DriverManager` abstraction ↔ vendor drivers), SLF4J API ↔ backends.

### Composite
**Intent:** compose objects into trees; treat a leaf and a group uniformly.
**Flight example:** a `PriceComponent` interface with `total()`; a `BaseFare` is a leaf, a `Bundle` (fare + bags + seat) is a composite holding other components. `bundle.total()` just sums children.

```java
interface PriceComponent { Money total(); }
record BaseFare(Money amount) implements PriceComponent { public Money total() { return amount; } }
record Bundle(List<PriceComponent> parts) implements PriceComponent {
    public Money total() { return parts.stream().map(PriceComponent::total).reduce(Money.ZERO, Money::add); }
}
```

- **Use when:** part-whole hierarchies — file systems, UI widget trees, org charts, expression trees, menus.
- **Trade-off:** hard to restrict what a composite may contain; leaf and composite sharing one interface can force meaningless methods on leaves.

### Decorator
**Intent:** attach extra behaviour to an object dynamically by wrapping it; same interface in, same interface out.

```java
interface Fare { Money price(); }
class BaseFare implements Fare { public Money price() { return base; } }
class BaggageFee implements Fare {
    private final Fare inner;
    BaggageFee(Fare inner) { this.inner = inner; }
    public Money price() { return inner.price().add(BAG); }
}
Fare f = new BaggageFee(new SeatFee(new BaseFare(...)));   // stackable at runtime
```

- **Use when:** you want combinations of optional behaviours without a subclass per combination.
- **Trade-off:** many tiny classes; identity breaks (`f != inner`); order of wrapping can matter.
- **In the wild:** `BufferedInputStream(new FileInputStream(..))`, `Collections.unmodifiableList`, servlet filters, Spring AOP proxies.

### Facade (object)
**Intent:** one simple interface over a complex subsystem.
**Flight example:** `BookingFacade.book(request)` internally calls inventory, pricing, payment, ticketing and notification. Controllers see one method.

- **Use when:** you want to reduce coupling to a subsystem or provide a coarse-grained API.
- **Trade-off:** it can become a god-object; it should *delegate*, not hold the logic.
- **Scales up:** API Gateway and BFF are facades at network scale ([API Gateway](05-distributed-communication-resilience-and-scaling.md#api-gateway), [BFF](05-distributed-communication-resilience-and-scaling.md#backend-for-frontend-bff)); Spring's service layer plays the same role inside an app.

### Flyweight
**Intent:** share immutable state between many objects to save memory.
**Flight example:** seat maps with 300 seats × 10,000 flights — the *intrinsic* state (seat type, legroom, cabin) is shared; only the *extrinsic* state (is it taken, by whom) is stored per flight.

- **Use when:** very large numbers of similar objects and memory is the problem (profiled, not guessed).
- **In the wild:** `Integer.valueOf` cache (-128…127), `String` intern pool, glyph objects in text editors.

### Proxy (object)
**Intent:** a stand-in with the same interface that controls access to the real object.

| Proxy type | What it does | Example |
|---|---|---|
| **Virtual** | Lazy-load an expensive object | Hibernate lazy collections |
| **Protection** | Check permissions first | `@PreAuthorize` |
| **Remote** | Hide that the object is across a network | Feign client, gRPC stub, RMI stub |
| **Caching** | Return stored result | `@Cacheable` |
| **Logging / smart reference** | Add cross-cutting behaviour | Spring AOP `@Transactional` |

- **vs Decorator:** same shape, different *intent*. Decorator *adds* behaviour the client chooses; Proxy *controls access* and usually manages the lifecycle of the target itself.
- **Gotcha:** Spring's proxies are why a `@Transactional` method called from *within the same class* is not transactional (the call bypasses the proxy).
- **Java mechanisms:** JDK dynamic proxy (interfaces) vs CGLIB/ByteBuddy subclass (classes).

### Also worth knowing (structural extras)
- **Marker interface** — an empty interface that tags a class (`Serializable`). Largely replaced by annotations.
- **Module** — group related code behind a public API with private internals (Java modules, packages).
- **Twin / Private class data** — rarely needed; skip unless you see them.

---

## Part C — Behavioural patterns

### Strategy
**Intent:** a family of interchangeable algorithms behind one interface; the context picks one.

```java
interface PricingStrategy { Money price(Fare base, Passenger p); }
class StandardPricing implements PricingStrategy { ... }
class LoyaltyPricing  implements PricingStrategy { ... }
class SurgePricing    implements PricingStrategy { ... }

class PriceService {
    Money quote(Fare f, Passenger p, PricingStrategy s) { return s.price(f, p); }
}
```

- **Use when:** you have an `if/else` or `switch` choosing between algorithms; behaviour varies by config/customer/context.
- **Modern form:** a lambda — `Comparator` is Strategy. If the interface has one method, you often need no class at all.
- **Trade-off:** clients must know which strategy to pick (pair it with a Factory); overkill for two stable branches.
- **In the wild:** `Comparator`, `ThreadPoolExecutor.RejectedExecutionHandler`, Spring Security `PasswordEncoder`, `AuthenticationProvider`.

### Observer
**Intent:** a subject notifies registered observers when its state changes; neither knows concrete types.

```java
interface BookingListener { void onConfirmed(Booking b); }
class BookingService {
    private final List<BookingListener> listeners = new CopyOnWriteArrayList<>();
    void confirm(Booking b) { /* … */ listeners.forEach(l -> l.onConfirmed(b)); }
}
// listeners: EmailSender, LoyaltyPointsAwarder, AnalyticsTracker
```

- **Use when:** one change must trigger an open-ended set of reactions (decouples "what happened" from "who cares").
- **Trade-offs:** hidden control flow; notification order undefined; a slow or throwing listener affects the publisher (unless async); memory leaks if observers aren't deregistered.
- **Scales up:** this is pub/sub and domain events — in-process `ApplicationEventPublisher`, then a message broker across services ([integration](06-integration-patterns.md)).
- **In the wild:** Swing listeners, `PropertyChangeListener`, Spring `ApplicationEvent`, RxJava / Reactor / `java.util.concurrent.Flow`, DOM events.

### Command (object)
**Intent:** wrap a request as an object, so you can queue it, log it, retry it or undo it.

```java
interface Command { void execute(); void undo(); }
record CancelBooking(BookingRepo repo, BookingId id) implements Command {
    public void execute() { repo.markCancelled(id); }
    public void undo()    { repo.markActive(id); }
}
```

- **Use when:** undo/redo, task queues, audit of operations, macro recording, transactional behaviour, scheduling.
- **Scales up:** commands become *messages* on a queue; compensating transactions in a [saga](04-distributed-data-consistency-and-coordination.md#saga) are "undo" commands; CQRS's *command side*.
- **In the wild:** `Runnable`/`Callable`, `ExecutorService.submit`, GUI actions, job schedulers.
- **vs Strategy:** Strategy = *how* to do something (algorithm); Command = *what* to do, packaged as a value, possibly later.

### State
**Intent:** an object changes behaviour when its internal state changes, by delegating to a state object.
**Flight example:** `Booking` is `PENDING → CONFIRMED → TICKETED → CANCELLED`. `cancel()` means "release hold" when pending, "refund" when ticketed, "no-op" when already cancelled.

```java
interface BookingState { BookingState cancel(Booking b); BookingState pay(Booking b); }
class Pending   implements BookingState { /* pay → Confirmed, cancel → Cancelled */ }
class Ticketed  implements BookingState { /* cancel → refund, then Cancelled */ }
```

- **Use when:** big `switch(status)` blocks repeated across many methods; transitions have rules.
- **Trade-off:** one class per state; for simple lifecycles an `enum` with per-constant methods or a transition table is enough. For complex lifecycles use a state-machine library (Spring Statemachine) and persist the state explicitly.
- **vs Strategy:** same shape. In State the object *itself* swaps the strategy as it transitions, and the states know about each other.

### Template Method
**Intent:** a base class defines an algorithm's skeleton and defers specific steps to subclasses.

```java
abstract class ReportGenerator {
    public final void generate() { fetch(); transform(); render(); deliver(); }  // fixed order
    protected abstract void render();
    protected void deliver() { /* default */ }                                    // hook
}
```

- **Use when:** several classes share an algorithm's outline but differ in details; you want to enforce the order.
- **Trade-off:** inheritance-based — rigid, fragile base class. Strategy (composition) is more flexible; Template Method is *less* code when variation is small.
- **In the wild:** `AbstractList`, `HttpServlet.service → doGet/doPost`, Spring `JdbcTemplate`, `RestTemplate` (template + callback), JUnit lifecycle.
- **Hollywood principle:** "don't call us, we'll call you" — the framework calls your code.

### Chain of Responsibility
**Intent:** pass a request along a chain; each handler either handles it or passes it on.
**Flight example:** booking validation chain — `FraudCheck → SeatAvailability → PaymentValid → BusinessRules`. Each can reject or continue.

- **Use when:** more than one object may handle a request, and the handler isn't known up front; processing is a pipeline of independent steps.
- **Trade-off:** no guarantee anyone handles it; hard to debug long chains; order matters.
- **In the wild:** servlet `FilterChain`, Spring Security filter chain, Spring `HandlerInterceptor`, middleware in Express/ASP.NET, logging levels.
- **Scales up:** the [Pipes and Filters](06-integration-patterns.md) integration pattern.

### Iterator
**Intent:** sequential access to a collection's elements without exposing its structure.
- Built into the language (`Iterable`, `for-each`, `Stream`). Know it, but you will rarely write one — except when adding lazy or paged traversal (e.g. iterate all search results across API pages).

### Mediator (object)
**Intent:** a central object coordinates a group of objects so they don't reference each other.
- **Use when:** many-to-many interactions have become spaghetti (dialog widgets; chat rooms; air-traffic control — apt for flights).
- **Trade-off:** the mediator tends to become a god-object.
- **vs Observer:** Observer = one-to-many broadcast, publisher doesn't coordinate. Mediator = central coordinator that knows about participants.
- **Scales up:** message broker / [orchestrator](04-distributed-data-consistency-and-coordination.md#saga), Spring `ApplicationEventPublisher`.
- **In the wild:** `java.util.Timer`, MediatR (.NET), Spring MVC `DispatcherServlet`.

### Memento
**Intent:** capture an object's state so it can be restored later, without exposing its internals.
- **Use when:** undo, checkpoints, drafts ("save partially filled booking form").
- **Trade-off:** memory cost of snapshots; consider storing deltas.
- **Scales up:** snapshots in [event sourcing](04-distributed-data-consistency-and-coordination.md#event-sourcing), database savepoints.

### Visitor
**Intent:** add new operations to an object structure without modifying its classes, by "visiting" each element.
- **Use when:** a stable class hierarchy (e.g. an AST: `Fare`, `Tax`, `Discount`) and an ever-growing set of operations (price, print, export, validate).
- **Trade-off:** adding a *new element type* forces every visitor to change; double-dispatch is hard to read.
- **Modern replacement:** Java 21 **sealed types + pattern matching `switch`** gives the same result with far less ceremony and with exhaustiveness checking.

```java
sealed interface Charge permits BaseFare, Tax, Discount {}
Money amount(Charge c) { return switch (c) {
    case BaseFare b -> b.amount();
    case Tax t      -> t.amount();
    case Discount d -> d.amount().negate();
}; }
```

### Interpreter
**Intent:** represent a grammar as classes and evaluate sentences in it.
- **Use when:** a small, stable DSL — pricing rules, filter expressions, routing conditions.
- **Reality check:** for anything non-trivial use a parser generator (ANTLR) or an existing expression language (SpEL, MVEL, JSONata, CEL) rather than hand-writing one.

### Also worth knowing (behavioural extras)

| Pattern | One line | Where |
|---|---|---|
| **Null Object** | A do-nothing implementation instead of `null`. | `Collections.emptyList()`, `Optional.empty()`, a `NoOpNotifier`. Removes `if (x != null)` clutter. |
| **Specification** | A business rule as a combinable object: `isPremium.and(isAdult).not(isBlacklisted)`. | Also a [DDD tactical building block](../notes/ddd-and-clean-architecture.md); Spring Data `Specification<T>` for dynamic queries. |
| **Event Bus / Event Aggregator** | Central in-process publish/subscribe. | Guava `EventBus`, Spring events. Observer made global. |
| **Interceptor** | Hooks that run before/after a call. | Spring `HandlerInterceptor`, gRPC interceptors, AOP advice. |
| **Registry** | Look up shared services by key. | Spring `ApplicationContext`; Service Locator is its anti-pattern cousin. |

---

## Part D — Concurrency patterns

The GoF book has none of these, but an architect gets asked about them constantly.

| Pattern | Problem it solves | Java / typical home |
|---|---|---|
| **Thread Pool / Executor** | Thread creation is costly and unbounded threads kill the box. Reuse a bounded set of workers. | `ExecutorService`, Tomcat connector threads |
| **Producer–Consumer** | Decouple work generation from processing; smooth bursts with a buffer. | `BlockingQueue`; at network scale, a message queue |
| **Future / Promise** | Get a result later without blocking now. | `Future`, `CompletableFuture`, JS `Promise` |
| **Immutability / Thread confinement** | Avoid locks by not sharing mutable state. | Records, `final`, `ThreadLocal` (careful with pools) |
| **Read–Write Lock** | Many readers, rare writers. | `ReentrantReadWriteLock`, `StampedLock` |
| **Double-Checked Locking** | Lazy init without locking every read. | Needs `volatile`; prefer the Holder idiom |
| **Monitor Object** | One lock guards an object's state. | `synchronized` methods |
| **Active Object / Actor** | Each object owns its state and processes messages one at a time — no shared memory. | Akka, Erlang, Vert.x verticles |
| **Scheduler** | Run tasks later/periodically. | `ScheduledExecutorService`, Quartz |
| **Balking** | Skip an action if the object is in the wrong state. | "If already booking in progress, ignore" |
| **Fork–Join / Map–Reduce** | Split, process in parallel, combine. | `ForkJoinPool`, parallel streams |
| **Virtual threads (Java 21)** | Cheap threads make thread-per-request viable again. | `Executors.newVirtualThreadPerTaskExecutor()` |

The distributed twins of these (queue load-levelling, competing consumers, distributed locks)
are in [05 resilience and scaling](05-distributed-communication-resilience-and-scaling.md) (queue levelling, backpressure), [06 integration](06-integration-patterns.md#competing-consumers) (competing consumers) and [04 coordination](04-distributed-data-consistency-and-coordination.md#part-d--coordination) (distributed locks, leases).

---

## Level 3 — deep dive

### The confusable groups

| Confusable | How to tell them apart |
|---|---|
| **Adapter / Decorator / Proxy / Facade** | All wrap something. *Adapter* changes the interface. *Decorator* keeps the interface and **adds** behaviour. *Proxy* keeps the interface and **controls access**. *Facade* gives a **new, simpler** interface to **many** objects. |
| **Strategy / State** | Same structure. Strategy: the client picks it, strategies are independent. State: the object transitions itself, states know each other. |
| **Strategy / Template Method** | Both vary part of an algorithm. Strategy = composition, swap at runtime. Template = inheritance, fixed at compile time. |
| **Factory Method / Abstract Factory / Builder** | One product chosen by input / a *family* of products / one product assembled in many steps. |
| **Observer / Mediator** | Broadcast without knowing the receivers / central coordinator that knows every participant. |
| **Command / Strategy** | An action packaged as data to run later / an interchangeable algorithm. |
| **Composite / Decorator** | Both recursive. Composite aggregates many children; Decorator wraps exactly one. |
| **Chain of Responsibility / Decorator** | Chain: a handler may *stop* the request. Decorator: always delegates and adds. |

### Patterns that modern languages have absorbed

| Pattern | What replaced it |
|---|---|
| Strategy, Command, Template Method (small cases) | Lambdas and method references |
| Iterator | `for-each`, Streams |
| Observer | Reactive streams (`Flow`, Reactor), framework events |
| Singleton | DI container scopes |
| Prototype | Records + "wither" methods, copy constructors |
| Visitor | Sealed types + pattern-matching `switch` |
| Builder | Records with named parameters (other languages), Lombok |
| Factory | DI + `Supplier`/`Function` maps |

Lesson: a pattern compensates for a language limitation. Know the *problem* so you recognise it
whether or not you need the full class structure.

### Where each lives in the Java/Spring stack

| Pattern | Where you've already used it |
|---|---|
| Singleton | Spring beans (default scope) |
| Factory | `BeanFactory`, `List.of`, `LoggerFactory.getLogger` |
| Builder | `ResponseEntity.ok().body(..)`, `HttpRequest.newBuilder()` |
| Proxy | `@Transactional`, `@Cacheable`, `@Async`, Hibernate lazy loading |
| Decorator | `BufferedReader`, `HttpServletRequestWrapper` |
| Adapter | `HandlerAdapter`, `InputStreamReader` |
| Facade | `JdbcTemplate`, `KafkaTemplate` |
| Template Method | `JdbcTemplate`, `AbstractRoutingDataSource` |
| Strategy | `Comparator`, `PasswordEncoder`, `ViewResolver` |
| Observer | `ApplicationEventPublisher` / `@EventListener` |
| Chain of Responsibility | Servlet filters, Spring Security filter chain |
| Command | `Runnable`, `Callable` |
| Iterator | `Iterable`, `Stream` |
| Composite | Spring Boot `CompositeHealthContributor`, Swing/AWT `Container`, the DOM tree |

### Anti-patterns at this level
- **Singleton abuse** — a "global variable with a class name".
- **Pattern fever** — adding Strategy + Factory + Builder to something that needed two lines. The rule: introduce a pattern when the *second or third* variation appears, not before (rule of three).
- **God object** — a Facade or Mediator that absorbed all the logic.
- **Inheritance for reuse** — deep hierarchies; use composition.
- **Anemic objects** — data holders with logic scattered elsewhere (see [DDD note](../notes/ddd-and-clean-architecture.md)).
- **Service Locator** hiding dependencies (see [application internals](02-application-internals.md#dependency-injection--inversion-of-control)).

### How to use patterns in an architect conversation
1. **Name the force first** ("this varies by region") before naming the pattern.
2. Show the *smallest* structure that works; mention what you'd refactor to if a third variant appears.
3. Know the trade-off sentence for each: "Decorator gives flexible composition at the cost of many small classes and order sensitivity."

---

## Check yourself

1. A `@Transactional` method calls another `@Transactional` method in the *same* class and the second one does not start a new transaction. Which pattern explains it, and why?
2. `Booking` has 11 status checks scattered across 8 methods. Which pattern, and what's the cheaper alternative if the lifecycle is simple?
3. Draw the difference between Decorator and Proxy for a `Fare`. What is each one's *intent*?
4. Why does Java 21 reduce the need for Visitor?
5. You need `Email`, `SMS`, `Push` × `BookingConfirmation`, `DelayAlert`, `Refund`. Which pattern avoids 9 classes, and how many classes remain?
6. When is Singleton legitimately the right choice, and what do you use instead in a Spring app?
7. Which creational pattern would you use to create a consistent set of `TaxCalculator + PaymentRail + ConsentForm` for a region?
8. Explain Chain of Responsibility using the Spring Security filter chain.

Next: [02 Application internals →](02-application-internals.md)
