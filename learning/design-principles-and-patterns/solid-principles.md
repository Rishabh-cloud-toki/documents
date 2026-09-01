**SOLID Design Principles - Short Notes**

---

### 1. **S - Single Responsibility Principle (SRP)**
- A class should have only one reason to change.
- Each class should focus on a single responsibility.
- Promotes:
  - Better readability
  - Easier maintenance
  - Reduced bug risk

**Example:**
Don't combine invoice calculation and email sending in the same class.

---

### 2. **O - Open/Closed Principle (OCP)**
- Software should be **open for extension**, but **closed for modification**.
- Add new behavior without altering existing code.
- Achieved using:
  - Interfaces
  - Abstract classes
  - Polymorphism

**Example:**
Use a new class that implements an interface to extend behavior instead of modifying an existing one.

---

### 3. **L - Liskov Substitution Principle (LSP)**
- Subtypes should be substitutable for their base types.
- Inherited classes should not break expected behavior of the parent.

**Example:**
A `Square` should not inherit from a `Rectangle` if it changes the expected behavior (i.e., width != height).

---

### 4. **I - Interface Segregation Principle (ISP)**
- Clients shouldn't be forced to depend on interfaces they don't use.
- Prefer small, role-specific interfaces over large, general-purpose ones.

**Example:**
Break `MultiFunctionDevice` interface into `Printable`, `Scannable`, and `Faxable`.

---

### 5. **D - Dependency Inversion Principle (DIP)**
- High-level modules should not depend on low-level modules.
- Both should depend on abstractions.
- Abstractions should not depend on details.
- Promotes loose coupling and testability.

**Bad Example:**
```java
class MySQLDatabase {
    void connect() {}
}
class DataService {
    MySQLDatabase db = new MySQLDatabase();
}
```

**Good Example:**
```java
interface Database {
    void connect();
}
class MySQLDatabase implements Database {
    public void connect() {}
}
class DataService {
    private Database db;
    DataService(Database db) {
        this.db = db;
    }
}
```

---

These principles help in creating well-structured, maintainable, and scalable object-oriented software systems.

