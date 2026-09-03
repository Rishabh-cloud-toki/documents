# Domain modeling, DDD and clean architecture

Revision notes. Continues from the system design walkthrough, using the same flight booking example.

---

# Part 1 — Domain modeling

## What it is

Deciding what the business actually is — its words, its boundaries, its rules, its lifecycles — and writing that down.

Classes are one way to write it down, and the least interesting one from an architect's seat.

## Domain modeling is not only classes

A model shows up in at least six places:

| Where | Example from the flight system |
|---|---|
| **Language** | The glossary. Deciding *quote* and *fare* are different words was modeling, and no class existed yet |
| **Boundaries** | The context map. Modeling at system level rather than object level |
| **Behaviour** | A booking goes HELD → CONFIRMED → TICKETED and can only be refunded after capture. A lifecycle, not a class diagram |
| **Process** | The saga. Airlines really do hold, then confirm, then ticket. We copied the business, we did not invent it |
| **Contracts** | `BookingConfirmed` carrying passengers and segments but not payment details is a decision about what a confirmation *means* |
| **Data** | Cardinality, ownership, what is frozen versus live |

## Domain model is not the data model

The data model says what is stored. The domain model says what is true and what is allowed.

"A booking must have at least one passenger" is a domain rule that happens to appear as a cardinality constraint. "A booking cannot be refunded before payment is captured" is equally part of the model and has no representation in the schema at all — it lives in behaviour.

## Two common mistakes in notes on this topic

**"Each microservice owns one bounded context."** Backwards. A context *may* become a microservice. It may also be a module inside a larger deployable, or span several services. The context is logical; the service is physical.

**"DDD is system design."** DDD is one input into system design. It tells you what the business is and where the seams are. It says nothing about deployment, scaling, caching, infrastructure or DR.

---

# Part 2 — The two halves of DDD

**DDD in one line:** design the software around how the business actually works, using the business's own words.

| | Strategic | Tactical |
|---|---|---|
| Question | How do we carve up the business? | How do we model this one area? |
| Unit of work | Contexts, teams, systems | Classes, objects |
| Output | Glossary, context map, core/supporting/generic | Entities, aggregates, value objects, events |
| Owner | Architect | Tech lead, senior developers |
| Cost of getting it wrong | Re-architecture | Refactoring |

That last row is why strategic matters more at architect level. A badly modelled aggregate is a week of work. Badly drawn boundaries means two teams tripping over each other for two years.

---

# Part 3 — Strategic DDD

## Ubiquitous language

One agreed vocabulary per context, shared between business and engineering. If the business says "hold," the class is called `Hold`.

The artifact is the glossary. It is produced, not assumed.

## Bounded contexts

An area where each word has exactly one meaning. The strongest signal you are looking at a boundary is a word changing meaning across it.

## Core, supporting, generic

Where your competitive advantage lives, so you can put your best people there and buy the rest.

For the flight system: Booking and Search are core. Payment and Ticketing are supporting. Identity and notifications are generic — buy them.

## Context map with named relationships

The part people skip, and the one carrying the most information. The types worth knowing:

| Relationship | Meaning | Flight example |
|---|---|---|
| **Customer-supplier** | One side is upstream and gives orders | Booking to Payment |
| **Conformist** | You accept their model as-is, no leverage | The identity provider |
| **Anticorruption layer** | You translate at the border so their model does not leak in | Both GDS integrations |
| **Shared kernel** | Two contexts share a small piece of model. Powerful and dangerous — every change needs two teams | Use rarely |
| **Separate ways** | No integration at all. Duplicating a little can beat coupling | — |
| **Open host service** | You publish a stable contract many consumers use | The event catalogue |

## How you arrive at it — event storming

The method, which most notes on DDD leave out entirely.

1. Get the business people in a room.
2. Write every significant thing that happens, in past tense, on stickies.
3. Arrange them left to right in time order.
4. Add what triggers each one and what data it needs.
5. Clusters emerge — events sharing vocabulary and data cling together. Those are your candidate contexts.
6. Classify each as core, supporting or generic.

Half a day to two days. Output: context map plus glossary.

Step 6 is the one that impresses, because it is a prioritisation decision rather than a modelling exercise.

## Tests to run against a candidate context map

- **Transaction test.** Does any operation need atomic changes in two contexts? Then you cut through the middle of something.
- **Chattiness test.** Sketch the main flows and count boundary crossings. Six crossings in one flow means a wrong boundary.
- **Change test.** Take ten likely change requests. How many contexts does each touch? If most touch four, the boundaries do not match how the business evolves. The most useful of the set.
- **Ownership test.** Can one team own this end to end without waiting on another?
- **Sentence test.** One sentence per context, no "and."
- **Vocabulary test.** Does any term mean two things inside one context?

There is no formula that outputs a context map. You propose it from the domain conversation and your experience, then stress-test it. Say that honestly in an interview — it is a stronger answer than pretending a method exists.

---

# Part 4 — Tactical DDD

One idea underneath it: **put the rules on the object that owns them, and make illegal states impossible to create.**

## Entity vs value object

An **entity** has identity that survives change. Passenger 4471 is the same passenger after you fix the spelling of their name.

A **value object** is defined entirely by its values. ₹8,400 is ₹8,400; two of them are interchangeable. Money, airport code, date range, address.

Practical rule: make it a value object unless you have a reason not to. They are immutable, easy to test, and carry their own validation. `new Money(-500)` throws; a bare `BigDecimal` does not. Most codebases have too few value objects and too many raw strings.

## Aggregate — the one that matters

A cluster of objects that must be consistent together, with one entry point called the root. Load it whole, save it whole, and nothing outside reaches into its innards.

Booking is a root. Passenger, Segment and Hold live inside it.

**The real definition: an aggregate is a consistency boundary** — the set of things that must be true at the same instant. That is why the sizing rules are what they are:

- **One aggregate per transaction.** If saving needs two atomically, they are really one aggregate, or you need eventual consistency between them.
- **Reference other aggregates by ID, not by object.** Booking holds a `paymentId`, not a `Payment`. This rule is what stops aggregates silently merging into one graph.
- **Keep them small.** The classic failure is Customer as a root containing every order ever placed. Load that to change a phone number and you have pulled ten years of history.
- **Anything outside the boundary is eventually consistent** — and that is fine, if you decided it deliberately.

## Invariants

The rules the aggregate guarantees are always true. For Booking:

- One to nine passengers
- At least one segment
- Cannot confirm without a valid, unexpired hold
- Cannot refund before capture
- Total equals fares plus taxes

Enforced inside the aggregate, never by the caller.

## Domain events

Something meaningful that happened, past tense. `SeatsHeld`, `BookingConfirmed`, `HoldExpired`.

The most useful tactical block for an architect, because events are what cross boundaries. Entities and value objects stay inside a service; events are how services talk. The event catalogue is tactical DDD leaking usefully into architecture.

## Supporting cast

**Domain service** — logic spanning two aggregates and belonging to neither. Calculating a through-fare across segments from two carriers. If it fits inside one aggregate, it is not a domain service. A payment gateway wrapper is usually an application service, not a domain service.

**Repository** — one per aggregate root, not per table. `BookingRepository` returns whole bookings. There is no `PassengerRepository`, because passengers are not independently retrievable. It is a pattern at the edge of the domain, not a domain concept — the business has no idea what a repository is.

**Factory** — for construction complicated enough to have its own rules.

## The anti-pattern to be able to name

**Anemic domain model.** Entities with nothing but getters and setters, all logic in service classes. It looks like DDD because the names are right, and it is procedural code with extra ceremony.

The tell: `Booking` has thirty setters and `BookingService` is 900 lines. The consequence: rules scatter, so "20 minutes" ends up in four places and three of them are wrong after the first change.

Most Spring codebases are anemic.

## When not to bother

Tactical DDD costs effort and pays off where rules are complex. A CRUD-heavy admin screen does not need aggregates — plain entities and a service layer are correct there. Generic subdomains you buy rather than model.

Knowing where *not* to apply it is part of the skill.

---

# Part 5 — Hexagonal and clean architecture

## They are the same pattern

Hexagonal (Cockburn), Onion (Palermo) and Clean (Martin) are three descriptions of one idea with different diagrams and different ring counts.

**The shared rule:** dependencies point inward, and the domain declares its own interfaces.

The one real distinction is Clean's explicit **use case layer** between entities and adapters. Entities hold rules true across the whole business ("a booking has at least one passenger"). Use cases hold rules specific to one operation ("confirming means re-validate, then hold, then charge"). Hexagonal does not insist on that split.

## Two layers, not three

A common mistake is drawing Adapters → Ports → Domain as three stacked layers, as if ports are a middle layer with their own code.

They are not. **A port is an interface that lives inside the domain.** Two layers: domain (including its port interfaces) and adapters (implementations). The adapter depends on the domain. Nothing points outward.

## Ports come in two kinds

The most useful part of the pattern, and the part usually missed.

**Driving ports (inbound).** Interfaces the outside world calls to make the domain do something. `CreateBookingUseCase`, `SearchFlights`. The application layer **implements** these; the REST controller calls them.

**Driven ports (outbound).** Interfaces the domain calls to reach the outside world. `BookingRepository`, `SeatInventory`, `PaymentProcessor`. The domain **declares** these; infrastructure implements them.

That asymmetry is the whole pattern. Both sides depend on the domain, in opposite directions, which is what makes the domain testable with no framework at all.

## The interface goes where it is used, not where it is implemented

`BookingRepository` is defined in the domain package, not the infrastructure package. That is dependency inversion in one sentence. If the interface sits next to its implementation, nothing has been gained.

## The naming trap

Naming a port after its implementation defeats it. `KafkaEventPublisher` as a *port* name means Kafka has leaked into the domain. Call the port `EventPublisher`; the Kafka adapter implements it. Same for `JpaBookingRepository` versus `BookingRepository`.

## Spring specifics

- Keep `domain/` free of every framework annotation. It should compile against nothing but the JDK.
- **Keep JPA annotations off domain entities.** The moment `Booking` carries `@Entity` and `@Column`, the domain depends on Hibernate. Use a separate persistence model and map between them. More code, and it is the difference between the pattern working and being decorative.
- `@Service` and `@Transactional` in the application layer is the compromise most teams accept. Purists move them to configuration. Either is defensible — be able to say which you chose and why.
- Watch `@ComponentScan` so layers do not blur.

## Where it fits in the design sequence

Below everything in the system design walkthrough. Bounded contexts decide the services; hexagonal decides the internal structure of one service. It is an LLD-level choice, and different services in the same system can make it differently. Nobody outside a service can tell which you chose.

## On "have you used it"

The question is really "do you know when not to." A CRUD service gains nothing — you get three files where one would do. It earns its keep when domain rules are complex or external systems are volatile.

The GDS integration is the natural example. `SeatInventory` as a port, one adapter per GDS. Second GDS means a second adapter, and the domain does not change. That is the same anticorruption layer from the context map, implemented as an adapter — a good thing to point out, because it connects the strategic decision to the code structure.

---


# Part 6 — Worked example: clean architecture for the Booking service


Skeleton only. Signatures, and full bodies for the three or four places where the pattern actually shows up.

---

### Package structure

```
com.airline.booking
│
├── domain/                          ← no framework imports, anywhere
│   ├── model/
│   │   ├── Booking.java                 aggregate root
│   │   ├── Passenger.java               entity, inside the aggregate
│   │   ├── Segment.java                 entity, inside the aggregate
│   │   ├── Hold.java                    entity, inside the aggregate
│   │   ├── BookingId.java               value object
│   │   ├── Pnr.java                     value object
│   │   ├── Money.java                   value object
│   │   ├── FareToken.java               value object
│   │   └── BookingStatus.java           enum
│   ├── event/
│   │   ├── DomainEvent.java
│   │   ├── SeatsHeld.java
│   │   └── BookingConfirmed.java
│   ├── port/
│   │   ├── in/                          driving ports — the outside calls these
│   │   │   ├── CreateBookingUseCase.java
│   │   │   └── ConfirmBookingUseCase.java
│   │   └── out/                         driven ports — the domain calls these
│   │       ├── BookingRepository.java
│   │       ├── SeatInventory.java
│   │       ├── FareValidator.java
│   │       ├── PaymentProcessor.java
│   │       └── EventPublisher.java
│   └── exception/
│       ├── HoldExpiredException.java
│       ├── FareChangedException.java
│       └── IllegalBookingStateException.java
│
├── application/                     ← use cases. Orchestration only, no business rules
│   ├── CreateBookingService.java
│   └── ConfirmBookingService.java
│
├── adapter/
│   ├── in/web/
│   │   ├── BookingController.java
│   │   ├── CreateBookingRequest.java
│   │   └── BookingResponse.java
│   └── out/
│       ├── persistence/
│       │   ├── BookingPersistenceAdapter.java
│       │   ├── BookingJpaEntity.java        ← @Entity lives HERE, not in domain
│       │   ├── PassengerJpaEntity.java
│       │   └── BookingMapper.java
│       ├── gds/                             ← the anticorruption layer
│       │   ├── GdsSeatInventoryAdapter.java
│       │   ├── GdsFareValidatorAdapter.java
│       │   ├── GdsSoapClient.java
│       │   └── GdsResponseTranslator.java
│       ├── payment/
│       │   └── PaymentGatewayAdapter.java
│       └── messaging/
│           └── KafkaEventPublisher.java
│
└── config/
    └── BeanConfiguration.java
```

**Note on where ports live.** I've put them under `domain/port`. Many teams put them in an `application` package instead. Either is fine — what matters is that the interface sits with the code that *uses* it, never with the code that implements it.

---

### 1. Value objects

Small, immutable, self-validating. Most codebases have too few of these.

```java
package com.airline.booking.domain.model;

public record Money(BigDecimal amount, Currency currency) {

    public Money {
        if (amount.signum() < 0)
            throw new IllegalArgumentException("Money cannot be negative");
        if (amount.scale() > currency.getDefaultFractionDigits())
            throw new IllegalArgumentException("Too many decimal places for " + currency);
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
    }
}
```

```java
public record FareToken(String value, Instant expiresAt) {
    public boolean hasExpired(Instant now) {
        return now.isAfter(expiresAt);
    }
}

public record Pnr(String value) {
    public Pnr {
        if (!value.matches("[A-Z0-9]{6}"))
            throw new IllegalArgumentException("PNR must be 6 alphanumeric characters");
    }
}
```

`new Money(-500)` is now impossible. A `BigDecimal` field would have allowed it.

---

### 2. The aggregate root

The important file. Every business rule lives here.

```java
package com.airline.booking.domain.model;

public class Booking {

    private final BookingId id;
    private final List<Passenger> passengers;
    private final List<Segment> segments;
    private final Money total;

    private BookingStatus status;
    private Hold hold;
    private Pnr pnr;
    private String paymentReference;          // reference by ID, not object

    private final List<DomainEvent> events = new ArrayList<>();

    // ---------- construction ----------

    private Booking(BookingId id, List<Passenger> passengers,
                    List<Segment> segments, Money total) {
        this.id = id;
        this.passengers = List.copyOf(passengers);
        this.segments = List.copyOf(segments);
        this.total = total;
        this.status = BookingStatus.DRAFT;
    }

    /** The only way to create a booking. Invariants enforced up front. */
    public static Booking create(List<Passenger> passengers,
                                 List<Segment> segments,
                                 Money total) {
        if (passengers.isEmpty() || passengers.size() > 9)
            throw new IllegalBookingStateException("A booking carries 1 to 9 passengers");
        if (segments.isEmpty())
            throw new IllegalBookingStateException("A booking needs at least one segment");

        return new Booking(BookingId.generate(), passengers, segments, total);
    }

    // ---------- behaviour ----------

    public void applyHold(Hold hold) {
        if (status != BookingStatus.DRAFT)
            throw new IllegalBookingStateException("Only a draft booking can be held");

        this.hold = hold;
        this.status = BookingStatus.HELD;
        raise(new SeatsHeld(id, hold.expiresAt(), segments));
    }

    public void confirm(Pnr pnr, String paymentReference, Instant now) {
        if (status != BookingStatus.HELD)
            throw new IllegalBookingStateException("Only a held booking can be confirmed");
        if (hold.hasExpired(now))
            throw new HoldExpiredException(hold.expiresAt());

        this.pnr = pnr;
        this.paymentReference = paymentReference;
        this.status = BookingStatus.CONFIRMED;
        raise(new BookingConfirmed(id, pnr, passengers, segments, total));
    }

    public void markTicketed() {
        if (status != BookingStatus.CONFIRMED)
            throw new IllegalBookingStateException("Only a confirmed booking can be ticketed");
        this.status = BookingStatus.TICKETED;
    }

    public boolean isRefundable() {
        return status == BookingStatus.CONFIRMED || status == BookingStatus.TICKETED;
    }

    // ---------- events ----------

    private void raise(DomainEvent event) { events.add(event); }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> copy = List.copyOf(events);
        events.clear();
        return copy;
    }

    // getters only — no setStatus(), no setPnr()
}
```

Four things worth noticing:

- **No setters for state.** The only route to `CONFIRMED` is `confirm()`, and that method checks the rules. No caller can skip them.
- **No Spring, no JPA, no Jackson.** This class compiles with nothing but the JDK. You can unit test it with no context, no database, no mocks.
- **`paymentReference` is a String, not a `Payment` object.** Payment is a different aggregate. Cross-aggregate references are by ID.
- **Events are raised inside the aggregate**, collected by the caller after the transaction commits.

---

### 3. Ports

#### Driving ports (inbound) — what the outside world can ask for

```java
package com.airline.booking.domain.port.in;

public interface CreateBookingUseCase {
    BookingId create(CreateBookingCommand command);
}

public record CreateBookingCommand(
        String itineraryId,
        FareToken fareToken,
        List<PassengerDetails> passengers,
        ContactDetails contact,
        String idempotencyKey) { }
```

```java
public interface ConfirmBookingUseCase {
    void confirm(BookingId bookingId, String paymentToken, String idempotencyKey);
}
```

#### Driven ports (outbound) — what the domain needs from the world

```java
package com.airline.booking.domain.port.out;

public interface BookingRepository {
    Optional<Booking> findById(BookingId id);
    Optional<Booking> findByIdempotencyKey(String key);
    void save(Booking booking);
}

public interface SeatInventory {
    Hold hold(List<Segment> segments, List<Passenger> passengers, Duration duration);
    Pnr confirm(Hold hold);
    void release(Hold hold);
}

public interface FareValidator {
    ValidatedFare revalidate(String itineraryId, FareToken token);
}

public interface PaymentProcessor {
    PaymentResult authorise(Money amount, String paymentToken, String idempotencyKey);
    void capture(String paymentReference);
    void voidAuthorisation(String paymentReference);
}

public interface EventPublisher {
    void publish(List<DomainEvent> events);
}
```

**Nothing here says GDS, SOAP, Postgres, Kafka or Stripe.** `SeatInventory` is the domain's own word. If the second GDS arrives, or the airline switches to direct NDC connectivity, this interface does not change.

That is the anticorruption layer from the context map, expressed as code.

---

### 4. Use case (the application layer)

Orchestration only. It sequences the ports and lets the aggregate make the decisions.

```java
package com.airline.booking.application;

@Service
@Transactional
public class CreateBookingService implements CreateBookingUseCase {

    private final FareValidator fareValidator;
    private final SeatInventory seatInventory;
    private final BookingRepository bookingRepository;
    private final EventPublisher eventPublisher;

    // constructor injection

    @Override
    public BookingId create(CreateBookingCommand command) {

        // idempotency — NFR-5
        Optional<Booking> existing =
                bookingRepository.findByIdempotencyKey(command.idempotencyKey());
        if (existing.isPresent()) return existing.get().id();

        // 1. re-validate the fare against the source (ADR-002)
        ValidatedFare fare = fareValidator.revalidate(
                command.itineraryId(), command.fareToken());

        if (fare.hasChanged())
            throw new FareChangedException(fare.oldAmount(), fare.newAmount());

        // 2. build the aggregate — invariants enforced inside
        Booking booking = Booking.create(
                toPassengers(command.passengers()),
                fare.segments(),
                fare.total());

        // 3. hold the seats — 20 minutes, FR-4
        Hold hold = seatInventory.hold(
                booking.segments(), booking.passengers(), Duration.ofMinutes(20));

        booking.applyHold(hold);

        // 4. persist and publish
        bookingRepository.save(booking);
        eventPublisher.publish(booking.pullEvents());

        return booking.id();
    }
}
```

Note what is **not** here: no rule about passenger counts, no check that a draft can be held, no PNR format validation. All of that is inside `Booking`. If a rule appears in this class, it is in the wrong place.

`@Service` and `@Transactional` are the compromise most teams accept — Spring annotations in the application layer but never in `domain/`. Purists move them to configuration. Either is defensible; be able to say which you chose.

---

### 5. Adapters

#### Inbound — REST

```java
package com.airline.booking.adapter.in.web;

@RestController
@RequestMapping("/v1/bookings")
public class BookingController {

    private final CreateBookingUseCase createBooking;   // depends on the PORT

    @PostMapping
    public ResponseEntity<BookingResponse> create(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody CreateBookingRequest request) {

        BookingId id = createBooking.create(request.toCommand(idempotencyKey));
        return ResponseEntity.status(CREATED).body(BookingResponse.of(id));
    }

    @ExceptionHandler(FareChangedException.class)
    ResponseEntity<ErrorResponse> handle(FareChangedException e) {
        return ResponseEntity.status(CONFLICT)
                .body(new ErrorResponse("FARE_CHANGED", e.getMessage()));
    }
}
```

The controller knows about the use case interface and nothing else. Swap REST for gRPC and only this package changes.

#### Outbound — persistence

Two models, and a mapper between them. This is the part teams skip, and skipping it is what quietly couples the domain to Hibernate.

```java
package com.airline.booking.adapter.out.persistence;

@Entity
@Table(name = "booking")
class BookingJpaEntity {                       // package-private, never leaves this package
    @Id UUID id;
    String pnr;
    @Enumerated(STRING) BookingStatus status;
    BigDecimal totalAmount;
    String currency;
    @OneToMany(cascade = ALL) List<PassengerJpaEntity> passengers;
    // ...
}

@Component
class BookingPersistenceAdapter implements BookingRepository {

    private final BookingJpaRepository jpa;
    private final BookingMapper mapper;

    @Override
    public Optional<Booking> findById(BookingId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public void save(Booking booking) {
        jpa.save(mapper.toJpa(booking));
    }
}
```

`Booking` has no `@Entity`. That is the whole point. The cost is a mapper; the benefit is that a Hibernate upgrade cannot break your business rules, and your domain tests need no database.

#### Outbound — the GDS anticorruption layer

The clearest example of why this pattern earns its keep.

```java
package com.airline.booking.adapter.out.gds;

@Component
class GdsSeatInventoryAdapter implements SeatInventory {

    private final GdsSoapClient client;
    private final GdsResponseTranslator translator;

    @Override
    public Hold hold(List<Segment> segments, List<Passenger> passengers, Duration duration) {

        // our model → their model
        GdsAvailabilityRequest request = translator.toGdsRequest(segments, passengers, duration);

        GdsAvailabilityResponse response;
        try {
            response = client.checkAndHold(request);
        } catch (GdsSoapFault fault) {
            // their error vocabulary → ours
            throw translator.toDomainException(fault);
        }

        // their model → our model
        return translator.toHold(response);
    }
}
```

Everything ugly about the GDS is contained in this package: SOAP envelopes, their three-letter status codes, their date format, their habit of returning success with an embedded error. `GdsResponseTranslator` is where that mess is converted, and it is the only place that knows about it.

When the second GDS arrives, you write `SecondGdsSeatInventoryAdapter implements SeatInventory` and change one line of configuration. The domain, the use cases and the controller are untouched.

---

### 6. What the test looks like

The payoff, and the thing to mention in an interview.

```java
class BookingTest {

    @Test
    void cannotConfirmAfterHoldExpires() {
        Booking booking = Booking.create(onePassenger(), oneSegment(), inr(8400));
        booking.applyHold(new Hold("GDS-REF", now().plusMinutes(20)));

        assertThrows(HoldExpiredException.class,
                () -> booking.confirm(new Pnr("ABC123"), "pay-1", now().plusMinutes(21)));
    }

    @Test
    void cannotCreateWithTenPassengers() {
        assertThrows(IllegalBookingStateException.class,
                () -> Booking.create(tenPassengers(), oneSegment(), inr(84000)));
    }
}
```

No Spring context. No database. No mocks. Runs in milliseconds. Every business rule in the system is testable this way, because every business rule lives in a plain Java class.

Use case tests need mocks for the ports, but only the ports:

```java
@Test
void rejectsWhenFareChanged() {
    when(fareValidator.revalidate(any(), any())).thenReturn(fareChangedBy(500));

    assertThrows(FareChangedException.class, () -> service.create(command));
    verifyNoInteractions(seatInventory);        // no seats held on a changed fare
}
```

---

### The dependency rule, checked

Every arrow points inward:

| Package | Depends on | Never depends on |
|---|---|---|
| `domain/model` | Nothing but the JDK | Everything else |
| `domain/port` | `domain/model` | Adapters, Spring, application |
| `application` | `domain/*` | Adapters |
| `adapter/in/web` | `domain/port/in` | Other adapters |
| `adapter/out/*` | `domain/port/out`, `domain/model` | Other adapters, application |
| `config` | Everything | — |

Worth enforcing with ArchUnit rather than discipline:

```java
@ArchTest
static final ArchRule domain_depends_on_nothing =
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..adapter..", "..application..",
                            "org.springframework..", "jakarta.persistence..");
```

One test, and the architecture stops being a diagram nobody enforces.

---

### When not to do this

The honest half of the answer.

This structure costs roughly three extra files per concept and a mapper you have to maintain. It pays for itself when the business rules are complex or the external systems are volatile — which describes Booking exactly, sitting between a legacy GDS and a payment gateway.

It does not pay for itself on a CRUD service. A reference-data service that returns airport codes should be a controller, a repository and an entity. Applying this pattern there produces ceremony with no benefit.

Different services in the same system can make this choice differently. It is an internal structure decision, one level below the container diagram, and nobody outside the service can tell which you chose.
