# 2.3 Domain-Driven Design (DDD)

## 2.3.1 What Is Domain-Driven Design?

**Domain-Driven Design (DDD)** is a software development approach that puts the business domain—the real-world problems, rules, and processes—at the heart of software design. Instead of starting with technical concerns like databases or frameworks, DDD starts with understanding the business.

### The Core Principle

> **Software should reflect how the business works, not the other way around.**

Instead of building software that forces the business to adapt to technical limitations, DDD builds software that expresses business concepts naturally and clearly.

## 2.3.2 Why Does DDD Exist?

Many software projects fail or become unmaintainable not because of bad code, but because the software doesn't accurately represent the business it's supposed to support. Let's look at a concrete example.

### Example: The Order Status Problem

Consider an e-commerce system where orders have different statuses: `Pending`, `Paid`, `Shipped`, `Delivered`, `Cancelled`. In a traditional approach, you might see code like this:

```javascript
// Traditional approach - scattered business logic
function updateOrderStatus(orderId, status) {
  const order = database.getOrder(orderId);
  order.status = status;
  database.save(order);
}
```

This code has serious problems:

- It allows impossible transitions (like going from `Delivered` back to `Pending`)
- Business rules are not enforced (you can cancel a shipped order)
- The code doesn't communicate business intent (what does `status` mean?)
- There's no place to add related logic (like sending confirmation emails)

### The DDD Solution

DDD addresses these issues by putting business rules and concepts into the code structure itself. Here's how the same scenario might look with DDD principles:

```javascript
// DDD approach - business rules in the domain
class Order {
  constructor(id) {
    this.id = id;
    this.status = OrderStatus.PENDING;
  }

  markAsPaid() {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error("Only pending orders can be paid");
    }
    this.status = OrderStatus.PAID;
    this.events.push(new OrderPaidEvent(this.id));
  }

  ship() {
    if (this.status !== OrderStatus.PAID) {
      throw new Error("Only paid orders can be shipped");
    }
    this.status = OrderStatus.SHIPPED;
    this.shippedAt = new Date();
  }

  cancel() {
    if (this.status === OrderStatus.SHIPPED) {
      throw new Error("Cannot cancel shipped orders");
    }
    this.status = OrderStatus.CANCELLED;
  }
}
```

Notice how the code now:

- Enforces business rules (you can't ship an unpaid order)
- Uses business language (`ship`, `markAsPaid`, `cancel`)
- Makes intentions clear and prevents mistakes
- Provides a natural place for domain events and related logic

## 2.3.3 When Should You Use DDD?

DDD is powerful but adds complexity. It's not always the right choice.

### ✅ Use DDD When:

- **Business rules are complex** (like pricing calculations, workflow approvals, inventory management)
- **Requirements change frequently** and you need flexibility
- **The project is long-lived** (not a prototype or quick tool)
- **Multiple teams** need to work together
- **Domain experts** (business people who understand the rules) are available

**Examples:** E-commerce platforms, banking systems, healthcare applications, supply chain management

### ❌ Avoid DDD When:

- The application is **simple CRUD** (basic data entry and display)
- **Business rules are minimal** or straightforward
- You need to **ship something quickly** (MVP or prototype)
- The team is **unfamiliar with DDD** and there's no time to learn
- The project is **short-lived**

**Examples:** Simple to-do apps, content-only websites, basic admin dashboards, one-off tools

---

## 2.3.4 Tactical Design: Implementing the Domain Model

Before we dive into the specific building blocks (entities, value objects, etc.), let's understand what **Tactical Design** means in DDD and how it relates to the bigger picture.

### What Is Tactical Design?

**Tactical Design** is about **how to implement domain concepts in code**. While Strategic Design (covered later) deals with organizing large systems into contexts and subdomains, Tactical Design focuses on modeling the domain within a single bounded context.

Think of it this way:

- **Strategic Design** = Architecture decisions (dividing the system)
- **Tactical Design** = Implementation decisions (writing the code)

### The Two Levels of DDD

```
DDD Design Levels
═════════════════

┌─────────────────────────────────────────────────┐
│         STRATEGIC DESIGN                        │
│         (Covered in 2.3.10-2.3.13)              │
│                                                 │
│  Organizing the System:                         │
│  • Which bounded contexts exist?                │
│  • How do contexts relate?                      │
│  • What are the subdomains?                     │
│  • Where to invest effort?                      │
│                                                 │
│  Scale: Entire system or organization           │
└─────────────────────────────────────────────────┘
                      ▲
                      │ Applied first to understand
                      │ the big picture
                      │
┌─────────────────────────────────────────────────┐
│         TACTICAL DESIGN                         │
│         (This section: 2.3.4-2.3.9)             │
│                                                 │
│  Implementing Within a Context:                 │
│  • How to model domain concepts?                │
│  • How to structure the code?                   │
│  • How to enforce business rules?               │
│  • How to protect invariants?                   │
│                                                 │
│  Scale: Within a single bounded context         │
└─────────────────────────────────────────────────┘
```

### Why Tactical Design Matters

Even with perfect strategic design, you can still end up with bad code inside each context. Tactical design provides patterns to:

1. **Make code match the business** - Classes and methods use domain language
2. **Enforce business rules** - Domain logic lives in domain objects, not scattered in services
3. **Protect data integrity** - Aggregates maintain consistency
4. **Separate concerns** - Domain logic stays independent of databases and UI

### The Tactical Patterns

DDD provides several tactical patterns (building blocks) for implementing domain models:

| Pattern            | Purpose                  | Example                      |
| ------------------ | ------------------------ | ---------------------------- |
| **Entity**         | Object with identity     | Customer, Order, Product     |
| **Value Object**   | Object defined by values | Money, Address, DateRange    |
| **Aggregate**      | Consistency boundary     | Order (with OrderItems)      |
| **Repository**     | Access to aggregates     | OrderRepository              |
| **Domain Service** | Multi-object operations  | PricingService               |
| **Factory**        | Complex object creation  | OrderFactory                 |
| **Domain Event**   | Something that happened  | OrderPlaced, PaymentReceived |

### Common Mistake: Tactical Without Strategic

Applying tactical patterns (entities, repositories) without strategic design leads to:

- ❌ One giant model trying to serve everyone
- ❌ "Customer" means 12 different things in different parts of the code
- ❌ Unclear boundaries and responsibilities
- ❌ Tight coupling between different business areas

**Both levels are essential.** Tactical design without strategic design is like having beautiful furniture arranged in a house with no walls—it's a mess.

### When to Use Tactical Patterns

**Use tactical patterns when:**

- ✅ You have complex business logic within a bounded context
- ✅ Business rules need to be explicit and testable
- ✅ The domain model changes frequently
- ✅ Multiple developers need to understand the domain

**Skip tactical patterns when:**

- ❌ The context is simple CRUD with minimal logic
- ❌ It's a generic subdomain (just integrate a library)
- ❌ The model is trivial and won't evolve

### Example: Where Tactical Design Lives

In our e-commerce system with multiple bounded contexts:

```
E-Commerce System
├── Sales Context (Bounded Context)
│   └── Tactical Design:
│       ├── Entities: Customer, Order, ShoppingCart
│       ├── Value Objects: Money, OrderStatus
│       ├── Aggregates: Order (root) + OrderItems
│       ├── Repositories: OrderRepository, CartRepository
│       └── Domain Services: CheckoutService
│
├── Inventory Context (Bounded Context)
│   └── Tactical Design:
│       ├── Entities: Product, StockLevel
│       ├── Value Objects: SKU, WarehouseLocation
│       ├── Aggregates: InventoryProduct + StockLevels
│       └── Repositories: InventoryRepository
│
└── Pricing Context (Bounded Context)
    └── Tactical Design:
        ├── Entities: PriceRule, Discount
        ├── Value Objects: Price, Percentage
        └── Domain Services: PriceCalculator
```

Each bounded context uses tactical patterns to implement its own domain model.

---

Now let's explore each tactical pattern in detail, starting with Entitie

---

## 2.3.5 Core Building Blocks of DDD

DDD provides several fundamental building blocks for modeling a domain. Let's explore each one using our e-commerce example throughout.

### 2.3.5.1 Entities: Objects with Identity

An **Entity** is an object that has a unique identity that persists over time, even if its properties change.

#### Key Characteristics:

- Has a unique identifier (ID)
- Can change state over time
- Identity is what matters, not the current state
- Usually represents a real-world concept that needs to be tracked

#### Example: Customer Entity

```javascript
class Customer {
  constructor(id, email, name) {
    this.id = id; // Unique identifier
    this.email = email;
    this.name = name;
    this.registeredAt = new Date();
    this.isVerified = false;
  }

  // Customer behavior
  updateEmail(newEmail) {
    if (!this.isValidEmail(newEmail)) {
      throw new Error("Invalid email format");
    }
    this.email = newEmail;
    this.isVerified = false; // Need to re-verify
  }

  verify() {
    this.isVerified = true;
  }

  isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

Even if the customer changes their email or name, they're still the same customer because their ID doesn't change.

#### Example: Order Entity

```javascript
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = "PENDING";
    this.createdAt = new Date();
    this.total = 0;
  }

  addItem(product, quantity) {
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify non-pending order");
    }

    this.items.push({
      productId: product.id,
      name: product.name,
      price: product.price,
      quantity: quantity,
    });

    this.recalculateTotal();
  }

  recalculateTotal() {
    this.total = this.items.reduce((sum, item) => {
      return sum + item.price * item.quantity;
    }, 0);
  }

  confirm() {
    if (this.items.length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    if (this.status !== "PENDING") {
      throw new Error("Order already confirmed");
    }
    this.status = "CONFIRMED";
  }
}
```

The Order entity encapsulates its own business rules: you can't add items after confirmation, you can't confirm an empty order, and the total is automatically recalculated.

---

### 2.3.5.2 Value Objects: Objects Defined by Their Values

A **Value Object** is defined only by its values, not by identity. Two value objects with the same values are considered equal.

#### Key Characteristics:

- No unique identifier
- Immutable (cannot be changed after creation)
- Equality based on values, not identity
- Describes or measures something

#### Example: Money Value Object

```javascript
class Money {
  constructor(amount, currency) {
    if (amount < 0) {
      throw new Error("Amount cannot be negative");
    }
    this.amount = amount;
    this.currency = currency;
    Object.freeze(this); // Make immutable
  }

  add(other) {
    if (this.currency !== other.currency) {
      throw new Error("Cannot add different currencies");
    }
    // Return NEW object, don't modify this one
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor) {
    return new Money(this.amount * factor, this.currency);
  }

  equals(other) {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString() {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }
}
```

#### Using the Money value object:

```javascript
const price1 = new Money(29.99, "USD");
const price2 = new Money(29.99, "USD");

console.log(price1.equals(price2)); // true - same values

const itemPrice = new Money(15.0, "USD");
const totalPrice = itemPrice.multiply(3);
console.log(totalPrice.toString()); // USD 45.00
```

#### Example: Address Value Object

```javascript
class Address {
  constructor(street, city, state, zipCode, country) {
    this.street = street;
    this.city = city;
    this.state = state;
    this.zipCode = zipCode;
    this.country = country;
    Object.freeze(this);
  }

  equals(other) {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.state === other.state &&
      this.zipCode === other.zipCode &&
      this.country === other.country
    );
  }

  toString() {
    return `${this.street}, ${this.city}, ${this.state} ${this.zipCode}, ${this.country}`;
  }
}
```

**Why immutable?** Because if you need to change an address, you create a new Address object. This prevents bugs where changing an address in one place accidentally affects it somewhere else.

---

### 2.3.5.3 Aggregates: Consistency Boundaries

An **Aggregate** is a cluster of related objects (entities and value objects) that are treated as a single unit. It has a boundary and a root entity. We have access to the aggregate through its root.

#### Key Concepts:

- **Aggregate Root:** The main entity that external code references
- **Boundary:** Defines what's inside vs outside the aggregate
- **Consistency:** The aggregate root enforces all business rules
- **Access Rule:** Everything inside must go through the root

#### Example: Order Aggregate

Let's build an Order aggregate that contains OrderItems. The Order is the aggregate root.

```javascript
// OrderItem - internal to the Order aggregate
class OrderItem {
  constructor(product, quantity) {
    if (quantity <= 0) {
      throw new Error("Quantity must be positive");
    }
    this.productId = product.id;
    this.productName = product.name;
    this.unitPrice = product.price;
    this.quantity = quantity;
  }

  getSubtotal() {
    return this.unitPrice * this.quantity;
  }

  updateQuantity(newQuantity) {
    if (newQuantity <= 0) {
      throw new Error("Quantity must be positive");
    }
    this.quantity = newQuantity;
  }
}

// Order - the Aggregate Root
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = []; // Collection of OrderItems
    this.status = "PENDING";
    this.createdAt = new Date();
  }

  // Public methods - the ONLY way to modify order items
  addItem(product, quantity) {
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }

    // Check if product already in order
    const existing = this.items.find((i) => i.productId === product.id);

    if (existing) {
      existing.updateQuantity(existing.quantity + quantity);
    } else {
      this.items.push(new OrderItem(product, quantity));
    }
  }

  removeItem(productId) {
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }
    this.items = this.items.filter((i) => i.productId !== productId);
  }

  updateItemQuantity(productId, newQuantity) {
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }

    const item = this.items.find((i) => i.productId === productId);
    if (!item) {
      throw new Error("Item not found in order");
    }
    item.updateQuantity(newQuantity);
  }

  getTotal() {
    return this.items.reduce((sum, item) => sum + item.getSubtotal(), 0);
  }

  confirm() {
    if (this.items.length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    if (this.status !== "PENDING") {
      throw new Error("Order already confirmed");
    }
    this.status = "CONFIRMED";
    this.confirmedAt = new Date();
  }
}
```

#### Why Aggregates Matter:

- External code can't directly access OrderItems—it must go through Order methods
- Order enforces all rules: no modifying confirmed orders, no empty orders
- Order maintains consistency: the total is always correct because items can only change through Order methods
- Clear boundary: everything related to order items is inside the Order aggregate

---

### 2.3.5.4 Repositories: Accessing Aggregates

A **Repository** provides a way to retrieve and store aggregates without exposing how they're stored.

#### Key Principles:

- One repository per aggregate root
- Collection-like interface (find, save, remove)
- Hides database/storage implementation
- Uses domain language in method names

#### Example: Order Repository

```javascript
class OrderRepository {
  constructor(database) {
    this.db = database;
  }

  // Find by ID
  async findById(orderId) {
    const data = await this.db.query("SELECT * FROM orders WHERE id = ?", [
      orderId,
    ]);

    if (!data) return null;

    // Reconstruct the Order aggregate from database data
    return this.hydrateOrder(data);
  }

  // Find by customer
  async findByCustomer(customerId) {
    const data = await this.db.query(
      "SELECT * FROM orders WHERE customer_id = ?",
      [customerId]
    );

    return data.map((d) => this.hydrateOrder(d));
  }

  // Save (create or update)
  async save(order) {
    const data = this.dehydrateOrder(order);

    await this.db.query(
      "INSERT INTO orders (id, customer_id, status, total) VALUES (?, ?, ?, ?) " +
        "ON DUPLICATE KEY UPDATE status = ?, total = ?",
      [
        data.id,
        data.customerId,
        data.status,
        data.total,
        data.status,
        data.total,
      ]
    );

    // Also save order items
    await this.saveOrderItems(order);
  }

  // Private methods to convert between domain objects and database
  hydrateOrder(data) {
    const order = new Order(data.id, data.customer_id);
    order.status = data.status;
    order.createdAt = data.created_at;
    // Load items...
    return order;
  }

  dehydrateOrder(order) {
    return {
      id: order.id,
      customerId: order.customerId,
      status: order.status,
      total: order.getTotal(),
    };
  }

  async saveOrderItems(order) {
    // Implementation to save order items...
  }
}
```

#### Using the Repository:

```javascript
// Application code doesn't know about database details
const orderRepo = new OrderRepository(database);

// Retrieve an order
const order = await orderRepo.findById("order-123");

// Modify it using domain methods
order.addItem(product, 2);
order.confirm();

// Save it back
await orderRepo.save(order);

// Find customer's orders
const customerOrders = await orderRepo.findByCustomer("customer-456");
```

#### Benefits:

- Domain code doesn't depend on database technology
- Easy to test with in-memory implementations
- Can switch databases without changing domain code
- Clear separation between domain logic and data access

---

## 2.3.6 Ubiquitous Language: Speaking the Same Language

One of the most important—and often overlooked—aspects of DDD is the **Ubiquitous Language**. This is a shared vocabulary that developers and domain experts (business people) use together when discussing the system.

### The Problem Without Ubiquitous Language

- Developers and business don't understand each other → requirements get misinterpreted
- Same concept gets different names (Customer/User/Client) → confusion everywhere
- Different concepts get the same name (Order meaning different things) → bugs and contradictions
- Code stops reflecting the business → logic becomes hidden behind technical terms
- Documentation and tests become unclear → only developers can guess what things mean
- The system drifts away from real domain rules → software becomes hard to maintain and easy to break

### The Solution: Ubiquitous Language

With DDD, everyone agrees on ONE set of terms and uses them consistently:

- In conversations
- In documentation
- In the code itself

To build a **ubiquitous language**:

- **Listen to the business**: use the exact words domain experts use daily.
- **Write a shared glossary**: define key terms clearly (and what they don't mean).
- **Remove synonyms**: don't call the same thing Customer/User/Client. Pick one.
- **Use the language in code**: class and method names should match business terms.
- **Model real actions**: `Invoice.markOverdue()` instead of `updateStatus()`.
- **Use domain events**: `OrderPlaced`, `PaymentReceived`, etc.
- **Split contexts when needed**: the same word can mean different things in different parts of the business.
- **Keep it consistent everywhere**: code, UI, APIs, tests, docs.

In short: **make business language the language of your software.**

### Example: Building a Ubiquitous Language for E-Commerce

Let's define terms clearly:

- **Shopping Cart:** The collection of items a customer intends to purchase
- **Order:** A confirmed purchase with payment information
- **Checkout:** The process of converting a cart into an order
- **Cart Abandonment:** When a cart has items but hasn't been converted to an order within 24 hours
- **Fulfillment:** The process of preparing and shipping an order

Now the code reflects this language:

```javascript
class ShoppingCart {
  // NOT 'Basket' or 'Session'
  addItem(product, quantity) {}
  removeItem(productId) {}
  checkout() {
    // Convert to Order
    return Order.fromCart(this, paymentInfo);
  }

  isAbandoned() {
    // Using business term
    const hoursSinceUpdate =
      (Date.now() - this.lastModified) / (1000 * 60 * 60);
    return hoursSinceUpdate > 24 && !this.isEmpty();
  }
}

class Order {
  static fromCart(cart, paymentInfo) {
    // Business process: checkout
  }

  beginFulfillment() {
    // Using business term
    this.status = "FULFILLING";
  }
}
```

### Benefits of Ubiquitous Language:

- Developers and business people understand each other perfectly
- Code is self-documenting—method names explain what they do
- New team members learn the business domain through the code
- Misunderstandings are caught early

---

## 2.3.7 Domain Services: When Behavior Doesn't Belong to One Object

Sometimes business logic doesn't naturally fit into a single entity or value object. This is where **Domain Services** come in. They perform domain operations that involve multiple objects or require external resources.

### When to Use Domain Services:

- The operation involves multiple aggregates
- The operation doesn't naturally belong to any single entity
- The operation represents a business process or calculation

### Example: Pricing Service

Calculating the final price of an order might involve:

- Base product prices
- Customer loyalty discounts
- Promotional codes
- Tax calculations based on shipping address
- Shipping costs

This logic doesn't belong in the Order entity alone—it needs information from multiple sources.

```javascript
class PricingService {
  constructor(taxCalculator, shippingCalculator) {
    this.taxCalculator = taxCalculator;
    this.shippingCalculator = shippingCalculator;
  }

  calculateOrderTotal(order, customer, shippingAddress, promoCode) {
    // Calculate base subtotal
    let subtotal = order.getItemsSubtotal();

    // Apply customer loyalty discount
    if (customer.loyaltyTier === "GOLD") {
      subtotal = subtotal * 0.9; // 10% discount
    }

    // Apply promotional code
    if (promoCode && promoCode.isValid()) {
      subtotal = promoCode.applyDiscount(subtotal);
    }

    // Calculate shipping
    const shipping = this.shippingCalculator.calculate(
      order.items,
      shippingAddress
    );

    // Calculate tax
    const tax = this.taxCalculator.calculate(subtotal, shippingAddress.state);

    return {
      subtotal: subtotal,
      shipping: shipping,
      tax: tax,
      total: subtotal + shipping + tax,
    };
  }
}

// Tax calculation - another domain service
class TaxCalculator {
  constructor(taxRates) {
    this.taxRates = taxRates; // e.g., { 'CA': 0.0725, 'NY': 0.04 }
  }

  calculate(amount, state) {
    const rate = this.taxRates[state] || 0;
    return amount * rate;
  }
}
```

### Using the Domain Service:

```javascript
// In application code
const pricingService = new PricingService(taxCalculator, shippingCalculator);

const order = await orderRepo.findById("order-123");
const customer = await customerRepo.findById(order.customerId);
const promoCode = await promoCodeRepo.findByCode("SUMMER2024");

const pricing = pricingService.calculateOrderTotal(
  order,
  customer,
  shippingAddress,
  promoCode
);

console.log(`Total: ${pricing.total}`);
```

### Important Distinctions:

- **Domain Service:** Contains business logic (like PricingService)
- **Application Service:** Coordinates workflows (like 'PlaceOrderService' that orchestrates the checkout process)
- **Infrastructure Service:** Technical concerns (like EmailService or PaymentGateway)

---

## 2.3.8 Tactical vs Strategic: Working Together

Now that we've covered the tactical building blocks (Entities, Value Objects, Aggregates, Repositories, Domain Services), let's see how tactical and strategic design work together.

### The Relationship

```
┌─────────────────────────────────────────────────────────┐
│  STRATEGIC DESIGN answers:                              │
│  "What bounded contexts do we have?"                    │
│  "How do they relate?"                                  │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Sales      │  │  Inventory   │  │   Shipping   │ │
│  │   Context    │◄─┤   Context    ├─►│   Context    │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                 │                  │          │
└─────────┼─────────────────┼──────────────────┼──────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│  TACTICAL DESIGN answers:                               │
│  "How do we implement each context?"                    │
│                                                         │
│  Sales Context:        Inventory Context:              │
│  • Order (Entity)      • Product (Entity)              │
│  • Money (VO)          • StockLevel (Entity)           │
│  • Order Aggregate     • Inventory Aggregate           │
│  • OrderRepository     • InventoryRepository           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Example: The Same Concept in Different Contexts

Let's see how "Product" is modeled differently using tactical patterns in different contexts:

```javascript
// CATALOG CONTEXT - Tactical implementation
class Product {
  // Entity
  constructor(id, name, description) {
    this.id = id; // Identity
    this.name = name;
    this.description = description;
    this.categories = [];
    this.images = [];
    this.searchKeywords = [];
  }

  updateDescription(newDescription) {
    this.description = newDescription;
    this.updateSearchKeywords();
  }

  addToCategory(category) {
    this.categories.push(category);
  }
}

// INVENTORY CONTEXT - Different tactical implementation
class Product {
  // Entity (but different model!)
  constructor(id, sku) {
    this.id = id; // Same ID, but different properties
    this.sku = sku;
    this.stockLevels = []; // Aggregate members
    this.reorderPoint = 0;
  }

  checkStockLevel() {
    const total = this.stockLevels.reduce(
      (sum, level) => sum + level.quantity,
      0
    );
    return total;
  }

  needsReorder() {
    return this.checkStockLevel() < this.reorderPoint;
  }
}

// SHIPPING CONTEXT - Yet another tactical implementation
class ShippableItem {
  // Different name, different focus
  constructor(productId, weight, dimensions) {
    this.productId = productId; // Reference to product
    this.weight = new Weight(weight); // Value Object
    this.dimensions = new Dimensions(dimensions); // Value Object
    this.fragile = false;
  }

  getShippingClass() {
    if (this.fragile) return "FRAGILE";
    if (this.weight.isHeavy()) return "HEAVY";
    return "STANDARD";
  }
}
```

**Key Insight:**

- **Strategic Design** says: "We have Catalog, Inventory, and Shipping contexts"
- **Tactical Design** says: "Within each context, here's how we model Product"

### When to Apply Each Level

```
Project Phase          | Strategic Design | Tactical Design
-----------------------|------------------|----------------
Initial planning       | ✓ Start here    | Wait
Understand domains     | ✓ Define contexts| Wait
Identify subdomains    | ✓ Map contexts   | Wait
Within each context    |                  | ✓ Now apply
Implement domain logic |                  | ✓ Use patterns
Refine over time       | ✓ Revisit        | ✓ Refactor
```

**Recommended Approach:**

1. **Start Strategic** - Identify bounded contexts and how they relate
2. **Then Tactical** - Model domain concepts within each context
3. **Iterate** - Refine both as you learn more about the domain

### Don't Skip Strategic Design

A common mistake is jumping straight to tactical patterns:

```javascript
// ❌ BAD: One "universal" Customer class serving all contexts
class Customer {
  // Sales properties
  loyaltyTier = "BRONZE";
  lifetimeValue = 0;

  // Shipping properties
  shippingAddresses = [];
  deliveryPreferences = {};

  // Support properties
  supportTickets = [];
  vipStatus = false;

  // Billing properties
  paymentMethods = [];
  invoices = [];

  // Marketing properties
  emailPreferences = {};
  campaignResponses = [];

  // ... this is a mess!
}
```

This "god class" happens when you skip strategic design and try to create one model for everything.

```javascript
// ✓ GOOD: Different models in different contexts

// Sales Context
class Customer {
  loyaltyTier;
  lifetimeValue;
}

// Shipping Context
class Recipient {
  shippingAddresses;
  deliveryPreferences;
}

// Support Context
class SupportCustomer {
  supportTickets;
  vipStatus;
}
```

Each context has a focused model using tactical patterns appropriately.

### Summary

**Strategic Design:**

- Divides the system into bounded contexts
- Identifies subdomains and priorities
- Defines how contexts integrate
- Prevents the "big ball of mud"

**Tactical Design:**

- Implements the model within each context
- Uses patterns: entities, value objects, aggregates
- Enforces business rules in code
- Keeps domain logic clean and testable

**Together they provide:**

- Clear system organization (strategic)
- Clean code implementation (tactical)
- Scalable, maintainable software

**Remember:** Tactical patterns are tools for building within a context. Strategic design determines which contexts you build and how they relate. You need both.

---

In the next sections, we'll explore Strategic Design in depth: Bounded Contexts, Subdomains, and Context Mapping.

---

## 2.3.9 Tactical Design Summary

Before moving to strategic design, let's summarize what we've learned about tactical patterns:

**Key Tactical Patterns:**

1. **Entities** - Objects with unique identity that can change over time
2. **Value Objects** - Immutable objects defined by their values
3. **Aggregates** - Consistency boundaries with an aggregate root
4. **Repositories** - Collection-like access to aggregates
5. **Domain Services** - Operations involving multiple objects
6. **Ubiquitous Language** - Shared vocabulary between developers and domain experts

**When to Apply Tactical Patterns:**

- ✅ Complex business logic within a bounded context
- ✅ Business rules need to be explicit and testable
- ✅ Domain model changes frequently
- ❌ Simple CRUD with minimal logic
- ❌ Generic subdomains (use libraries instead)

**Common Pitfalls:**

- Anemic domain models (entities with no behavior)
- Over-engineering simple features
- Exposing internal aggregate state
- Applying DDD without understanding the domain

Now let's explore Strategic Design, which helps us organize these tactical elements into larger systems.

---

## 2.3.10 Strategic Design: Organizing Complex Systems

So far, we've focused on **tactical patterns**—how to model individual domain concepts using entities, value objects, and aggregates. But what happens when your system grows? An e-commerce platform might have inventory management, order processing, customer accounts, shipping, marketing, and payment processing. How do you organize all of this?

This is where **strategic design** comes in. Strategic DDD provides patterns for dividing large systems into manageable pieces and defining how those pieces relate to each other.

---

## 2.3.11 Bounded Contexts: Clear Boundaries for Models

### What Is a Bounded Context?

A **Bounded Context** is an explicit boundary within which a particular domain model applies. Inside this boundary, all terms have specific, unambiguous meanings. Outside the boundary, the same terms might mean something completely different.

Think of it like countries with different languages: the word "gift" means "present" in English but "poison" in German. The country border is the boundary that determines which meaning applies.

### Why Bounded Contexts Matter

In a large e-commerce system, the word "Product" appears everywhere. But does it mean the same thing everywhere?

- **In the Catalog context:** Product = searchable item with description, images, categories, SEO metadata
- **In the Inventory context:** Product = stockable item with SKU, warehouse location, quantity on hand
- **In the Shipping context:** Product = shippable item with weight, dimensions, fragility
- **In the Pricing context:** Product = priceable item with base price, discount rules, tax category

These are **different models** of the same concept. Trying to create one "universal Product" that serves all these contexts leads to a bloated, confusing mess.

### Example: Customer in Different Contexts

Let's look at how "Customer" differs across contexts in our e-commerce system:

```javascript
// SALES CONTEXT - Customer as a buyer
class Customer {
  constructor(id, email, name) {
    this.id = id;
    this.email = email;
    this.name = name;
    this.loyaltyTier = "BRONZE";
    this.lifetimeValue = 0;
    this.preferredCategories = [];
  }

  upgradeToGoldTier() {
    if (this.lifetimeValue > 10000) {
      this.loyaltyTier = "GOLD";
    }
  }

  addPurchase(orderTotal) {
    this.lifetimeValue += orderTotal;
  }
}

// SHIPPING CONTEXT - Customer as a delivery recipient
class Recipient {
  constructor(id, fullName) {
    this.id = id;
    this.fullName = fullName;
    this.shippingAddresses = [];
    this.deliveryPreferences = {
      allowWeekendDelivery: true,
      requireSignature: false,
      safeDropLocation: null,
    };
  }

  addShippingAddress(address) {
    this.shippingAddresses.push(address);
  }

  getPreferredAddress() {
    return this.shippingAddresses.find((a) => a.isDefault);
  }
}

// SUPPORT CONTEXT - Customer as a support case owner
class SupportCustomer {
  constructor(id, contactEmail, phoneNumber) {
    this.id = id;
    this.contactEmail = contactEmail;
    this.phoneNumber = phoneNumber;
    this.supportTickets = [];
    this.communicationPreference = "EMAIL";
    this.vipStatus = false;
  }

  openTicket(issue) {
    const ticket = new SupportTicket(issue, this.id);
    this.supportTickets.push(ticket);
    return ticket;
  }
}
```

Notice:

- Same concept (customer) but **different properties** in each context
- Different **behaviors** based on what matters in that context
- Even different **names** (Customer vs Recipient vs SupportCustomer)

### Bounded Context Boundaries

A bounded context typically aligns with:

- **A team's responsibility** (the Shipping team owns the Shipping context)
- **A specific business capability** (everything related to order fulfillment)
- **A natural linguistic boundary** (where terms start meaning different things)

### Benefits of Bounded Contexts

1. **Models stay focused** - Each model only includes what's relevant to its context
2. **Teams can work independently** - Clear boundaries mean less coordination
3. **Language is precise** - No ambiguity within a context
4. **Easier to understand** - Smaller, focused models are simpler than one giant model

---

## 2.3.12 Subdomains: Understanding Business Complexity

While Bounded Contexts are about **technical organization**, Subdomains are about **business organization**. A subdomain represents a distinct area of business activity.

### Types of Subdomains

DDD distinguishes three types of subdomains:

#### 1. Core Subdomain

This is your **competitive advantage**—what makes your business unique and valuable.

**Characteristics:**

- Differentiates you from competitors
- Complex business rules
- Requires deep domain expertise
- Worth heavy investment
- Should be built in-house with best developers

**E-commerce Examples:**

- **Personalized recommendation engine** (like Amazon's "customers who bought this also bought")
- **Dynamic pricing algorithm** (adjusting prices based on demand, competition, inventory)
- **Fraud detection system** (unique ML models protecting your specific business)

#### 2. Supporting Subdomain

These are necessary for the business but **don't provide competitive advantage**. They support the core domain.

**Characteristics:**

- Important but not differentiating
- Simpler than core domain
- Can be custom-built with less investment
- Could potentially be outsourced

**E-commerce Examples:**

- **Warehouse management** (tracking which shelf products are on)
- **Review moderation** (flagging inappropriate reviews)
- **Gift wrapping service** (managing gift options)

#### 3. Generic Subdomain

These are **common to all businesses** in your industry. No competitive advantage here.

**Characteristics:**

- Well-understood problems
- Off-the-shelf solutions exist
- Don't waste time building these
- Buy or use open-source solutions

**E-commerce Examples:**

- **User authentication** (login, password reset)
- **Email notifications** (order confirmations, shipping updates)
- **Payment processing** (use Stripe, PayPal, etc.)
- **Shipping calculation** (use carrier APIs)

### Subdomain Strategy: Where to Invest

This is crucial for resource allocation:

```
┌─────────────────────────────────────────────────────────┐
│ INVESTMENT STRATEGY                                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  CORE SUBDOMAIN           → Best developers            │
│  (Your competitive        → Most time/money            │
│   advantage)              → Custom-built               │
│                           → Deep DDD modeling          │
│                                                         │
│  SUPPORTING SUBDOMAIN     → Good developers            │
│  (Necessary but not       → Moderate investment        │
│   differentiating)        → Custom or adapted          │
│                           → Simpler modeling           │
│                                                         │
│  GENERIC SUBDOMAIN        → Don't build!               │
│  (Common commodity)       → Buy/integrate              │
│                           → Minimal customization      │
│                           → Use existing solutions     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Example: E-Commerce Subdomains

Let's map out an e-commerce platform's subdomains:

**CORE SUBDOMAINS:**

- **Product Discovery** - AI-powered search and recommendations
- **Dynamic Pricing** - Real-time price optimization
- **Inventory Optimization** - Predictive stocking algorithms

**SUPPORTING SUBDOMAINS:**

- **Order Management** - Processing and tracking orders
- **Customer Loyalty** - Points, tiers, rewards programs
- **Warehouse Operations** - Pick, pack, ship processes
- **Returns Processing** - Handling product returns

**GENERIC SUBDOMAINS:**

- **Authentication** - User login/security (use Auth0, Okta)
- **Payment Processing** - Charging cards (use Stripe)
- **Email/SMS** - Notifications (use SendGrid, Twilio)
- **Shipping Calculation** - Carrier rates (use ShipStation API)

### Subdomains vs Bounded Contexts

Important distinction:

- **Subdomain** = Business/problem space ("what the business does")
- **Bounded Context** = Solution space ("how we organize the software")

Often there's a 1:1 mapping (one subdomain = one bounded context), but not always:

- **One subdomain might split into multiple contexts** (if it's very complex)
- **Multiple simple subdomains might share one context** (if they're closely related)

---

## 2.3.13 Context Mapping: How Contexts Relate

When you have multiple bounded contexts, they need to communicate. **Context Mapping** is about explicitly defining these relationships.

### Why Context Mapping Matters

Without explicit mapping, you get:

- Unclear dependencies between teams
- Tight coupling between contexts
- Integration nightmares
- Conflicting models polluting each other

Context maps make integration points visible and managed.

### Common Context Relationship Patterns

#### 1. Partnership

Two contexts work together, teams collaborate closely.

```
┌──────────────────┐         ┌──────────────────┐
│   Sales Context  │◄───────►│ Billing Context  │
│                  │  Equal  │                  │
│  - Orders        │  Partners│  - Invoices     │
│  - Customers     │         │  - Payments      │
└──────────────────┘         └──────────────────┘
```

**Characteristics:**

- Mutual dependency
- Teams coordinate releases
- Share development efforts
- Both succeed or fail together

**Example:** Sales and Billing teams work together to ensure orders are properly invoiced.

#### 2. Shared Kernel

Two contexts share a subset of the domain model.

```
┌──────────────────────────────────────────┐
│         Shared Kernel                    │
│    (Common Customer Model)               │
│  - Customer ID, Name, Contact           │
└───────────────┬──────────────────────────┘
                │
        ┌───────┴────────┐
        │                │
┌───────▼─────┐    ┌────▼────────┐
│   Sales     │    │  Support    │
│   Context   │    │  Context    │
│             │    │             │
│ + Purchase  │    │ + Tickets   │
│   History   │    │ + Issues    │
└─────────────┘    └─────────────┘
```

**Characteristics:**

- Shared code between contexts
- Changes need coordination
- Shared tests required
- Higher coupling (use sparingly)

**Example:** Sales and Support share basic Customer data structure.

⚠️ **Warning:** Shared Kernel creates coupling. Only use when teams are very closely aligned.

#### 3. Customer-Supplier

One context (supplier) provides services to another (customer). Supplier's priorities may not align with customer's needs.

```
┌──────────────────┐
│   Catalog        │  Supplier
│   (Upstream)     │  (defines API)
│                  │
│  - Product Info  │
└────────┬─────────┘
         │ API
         │
┌────────▼─────────┐
│   Order Mgmt     │  Customer
│   (Downstream)   │  (consumes API)
│                  │
│  - Order Items   │
└──────────────────┘
```

**Characteristics:**

- Upstream (supplier) has power
- Downstream (customer) depends on upstream
- Downstream can influence but not control
- Upstream team defines interfaces

**Example:** Order Management depends on Catalog for product information. Catalog team controls the product API.

#### 4. Conformist

Downstream context conforms to upstream model without negotiation.

```
┌──────────────────┐
│   External       │  Upstream
│   Payment API    │  (3rd party)
│   (Stripe/PayPal)│
└────────┬─────────┘
         │ Must conform
         │
┌────────▼─────────┐
│   Our Payment    │  Downstream
│   Integration    │  (no influence)
│                  │
└──────────────────┘
```

**Characteristics:**

- No negotiation possible
- Downstream accepts upstream model as-is
- Common with external/legacy systems
- Simplest integration (but least flexible)

**Example:** Integrating with Stripe's payment API—you use their model, period.

#### 5. Anti-Corruption Layer (ACL)

Downstream context protects itself from upstream by translating the upstream model.

```
┌──────────────────┐
│   Legacy         │  Upstream
│   Inventory      │  (old system)
│   System         │
└────────┬─────────┘
         │
    ┌────▼────────────────┐
    │ Anti-Corruption     │  Translation Layer
    │ Layer (Adapter)     │  (protects our model)
    └────┬────────────────┘
         │
┌────────▼─────────┐
│   Modern         │  Downstream
│   Warehouse      │  (clean model)
│   Management     │
└──────────────────┘
```

**Characteristics:**

- Protects downstream from upstream complexity
- Translates between models
- Allows clean domain model
- More work but prevents pollution

**Example:** Legacy inventory system uses confusing codes. ACL translates to clear domain model.

```javascript
// Anti-Corruption Layer Example
class InventoryACL {
  constructor(legacyInventoryService) {
    this.legacy = legacyInventoryService;
  }

  // Translate from legacy model to our domain model
  async getProductStock(productId) {
    // Legacy system uses cryptic format
    const legacyData = await this.legacy.queryStock({
      sku_cd: productId,
      loc_typ: "WH",
      stat_flg: "A",
    });

    // Translate to our clean domain model
    return new StockLevel({
      productId: productId,
      quantity: legacyData.qty_on_hnd,
      location: this.translateLocation(legacyData.loc_cd),
      availableForSale: legacyData.stat_flg === "A",
    });
  }

  translateLocation(legacyCode) {
    const locationMap = {
      WH01: "Main Warehouse",
      WH02: "East Coast Hub",
      WH03: "West Coast Hub",
    };
    return locationMap[legacyCode] || "Unknown";
  }

  // Translate from our model to legacy
  async reserveStock(productId, quantity) {
    return await this.legacy.reserveInventory({
      sku_cd: productId,
      qty_rsv: quantity,
      rsv_typ: "ORD",
      usr_id: "ECOM_SYS",
    });
  }
}
```

#### 6. Open Host Service (OHS)

Upstream defines a protocol/API that serves many different downstream consumers.

```
                    ┌──────────────────┐
                    │  Product Catalog │  Upstream
                    │  (Open Host)     │  (REST API)
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼───┐   ┌─────▼──────┐  ┌───▼─────────┐
    │ Order Mgmt  │   │ Warehouse  │  │  Marketing  │
    │ (Consumer)  │   │ (Consumer) │  │  (Consumer) │
    └─────────────┘   └────────────┘  └─────────────┘
```

**Characteristics:**

- Public, documented API
- Serves multiple consumers
- Versioned and stable
- Think REST/GraphQL APIs

**Example:** Product Catalog provides REST API consumed by Order Management, Warehouse, and Marketing.

#### 7. Published Language (PL)

A well-documented, shared data format between contexts.

```
┌──────────────────┐         ┌──────────────────┐
│   Order Context  │         │ Shipping Context │
└────────┬─────────┘         └─────────┬────────┘
         │                             │
         │      Published Language     │
         │      (JSON Schema/XML)      │
         └──────────────┬──────────────┘
                        │
                ┌───────▼────────┐
                │  Standard      │
                │  Order Format  │
                │  (Documented)  │
                └────────────────┘
```

**Characteristics:**

- Standardized format (JSON, XML, Protobuf)
- Well-documented schema
- Multiple contexts can use it
- Enables loose coupling

**Example:** Standard JSON format for order events that multiple contexts consume.

### Complete Context Map: E-Commerce System

Here's how these patterns work together in a real system:

```
                    E-COMMERCE SYSTEM CONTEXT MAP
                    ==============================

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌──────────────┐                    ┌──────────────┐        │
│   │   CATALOG    │◄──────Partnership─►│  INVENTORY   │        │
│   │   CONTEXT    │                    │   CONTEXT    │        │
│   │              │                    │              │        │
│   │ - Products   │                    │ - Stock      │        │
│   │ - Categories │                    │ - Warehouses │        │
│   └──────┬───────┘                    └──────┬───────┘        │
│          │                                   │                 │
│          │ OHS (REST API)                    │ OHS             │
│          │                                   │                 │
│   ┌──────▼───────────────────────────────────▼───────┐        │
│   │                                                   │        │
│   │            ORDER MANAGEMENT CONTEXT              │        │
│   │            (CORE SUBDOMAIN)                      │        │
│   │                                                   │        │
│   │  - Shopping Cart    - Order Processing           │        │
│   │  - Checkout         - Order Tracking             │        │
│   │                                                   │        │
│   └──────┬─────────────┬─────────────┬───────────────┘        │
│          │             │             │                         │
│          │Customer-    │Partnership  │ACL                      │
│          │Supplier     │             │                         │
│          │             │             │                         │
│   ┌──────▼───────┐ ┌──▼──────────┐ ┌▼────────────────┐       │
│   │   PRICING    │ │  SHIPPING   │ │  PAYMENT        │       │
│   │   CONTEXT    │ │  CONTEXT    │ │  INTEGRATION    │       │
│   │              │ │             │ │  (External)     │       │
│   │ - Dynamic    │ │ - Carriers  │ │                 │       │
│   │   Pricing    │ │ - Tracking  │ │ - Stripe API    │       │
│   │ - Discounts  │ │ - Delivery  │ │ - PayPal API    │       │
│   └──────────────┘ └─────────────┘ └─────────────────┘       │
│          ▲                                    │                │
│          │                                    │Conformist      │
│          │Shared Kernel                       │                │
│          │                                    │                │
│   ┌──────┴───────┐                    ┌──────▼────────┐       │
│   │   CUSTOMER   │                    │   BILLING     │       │
│   │   CONTEXT    │◄──────Partnership─►│   CONTEXT     │       │
│   │              │                    │               │       │
│   │ - Accounts   │                    │ - Invoices    │       │
│   │ - Loyalty    │                    │ - Refunds     │       │
│   └──────────────┘                    └───────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Legend:
-------
◄──Partnership──►  : Teams coordinate closely, mutual dependency
──OHS (API)──►    : Open Host Service, published API
──Customer-Supplier→ : Upstream team controls interface
──Shared Kernel──  : Shared code/model (use sparingly)
──ACL──►          : Anti-Corruption Layer protects downstream
──Conformist──►   : Downstream accepts upstream model as-is
```

### Benefits of Context Mapping

1. **Makes integration explicit** - No hidden dependencies
2. **Guides team organization** - Shows which teams need to coordinate
3. **Identifies risks** - Spots tight coupling early
4. **Supports planning** - Know who depends on whom
5. **Facilitates communication** - Common language for discussing integration

---

## 2.3.14 Putting It All Together: A Complete Strategic and Tactical Example

Let's rebuild our e-commerce checkout example with both strategic and tactical design. This time, we'll show how multiple bounded contexts work together using tactical patterns within each context.

### System Overview

Our e-commerce platform has these bounded contexts:

1. **Sales Context** (Core Subdomain) - Handles shopping cart and checkout
2. **Inventory Context** (Supporting Subdomain) - Manages stock levels
3. **Pricing Context** (Core Subdomain) - Dynamic pricing and discounts
4. **Shipping Context** (Supporting Subdomain) - Delivery management
5. **Payment Context** (Generic Subdomain) - External payment integration

### Context Map

```
    ┌──────────────┐
    │  INVENTORY   │
    │   CONTEXT    │
    └──────┬───────┘
           │ OHS
           │
    ┌──────▼───────────────────┐
    │                          │
    │    SALES CONTEXT         │
    │    (CORE - Checkout)     │
    │                          │
    └──┬────────┬──────────┬──┘
       │        │          │
       │Partner │Customer  │ACL
       │ship    │Supplier  │
       │        │          │
┌──────▼───┐ ┌─▼──────┐ ┌─▼───────────┐
│ PRICING  │ │SHIPPING│ │  PAYMENT    │
│ CONTEXT  │ │CONTEXT │ │  (External) │
└──────────┘ └────────┘ └─────────────┘
```

### Implementation

#### 1. Sales Context - The Core Domain

This is our competitive advantage: a smooth, optimized checkout experience.

```javascript
// ===== SALES CONTEXT (Bounded Context) =====

// This context's view of Customer (Entity)
class Customer {
  constructor(id, email, loyaltyTier) {
    this.id = id;
    this.email = email;
    this.loyaltyTier = loyaltyTier; // Used for pricing
  }
}

// Shopping Cart Aggregate
class ShoppingCart {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
  }

  addItem(productId, productName, basePrice, quantity) {
    const existing = this.items.find((i) => i.productId === productId);
    if (existing) {
      existing.quantity += quantity;
    } else {
      this.items.push({
        productId,
        productName,
        basePrice,
        quantity,
      });
    }
  }

  getItems() {
    return [...this.items];
  }

  clear() {
    this.items = [];
  }
}

// Order Aggregate (Aggregate Root)
class Order {
  constructor(id, customerId, shippingAddress) {
    this.id = id;
    this.customerId = customerId;
    this.shippingAddress = shippingAddress;
    this.items = [];
    this.pricing = null;
    this.status = "PENDING";
  }

  static fromCart(cart, shippingAddress) {
    const order = new Order(
      this.generateOrderId(),
      cart.customerId,
      shippingAddress
    );

    cart.getItems().forEach((item) => {
      order.items.push({
        productId: item.productId,
        productName: item.productName,
        quantity: item.quantity,
      });
    });

    return order;
  }

  setPricing(pricingDetails) {
    this.pricing = pricingDetails;
  }

  confirm() {
    if (!this.pricing) {
      throw new Error("Order must be priced before confirmation");
    }
    this.status = "CONFIRMED";
  }

  static generateOrderId() {
    return "ORD-" + Date.now();
  }
}

// Application Service - Orchestrates across contexts
class CheckoutService {
  constructor(
    cartRepo,
    orderRepo,
    customerRepo,
    inventoryService, // Calls Inventory Context
    pricingService, // Calls Pricing Context
    shippingService, // Calls Shipping Context
    paymentService // Calls Payment Context (external)
  ) {
    this.cartRepo = cartRepo;
    this.orderRepo = orderRepo;
    this.customerRepo = customerRepo;
    this.inventoryService = inventoryService;
    this.pricingService = pricingService;
    this.shippingService = shippingService;
    this.paymentService = paymentService;
  }

  async checkout(customerId, shippingAddress, paymentMethod) {
    // 1. Load cart and customer (from Sales Context)
    const cart = await this.cartRepo.findByCustomer(customerId);
    if (cart.getItems().length === 0) {
      throw new Error("Cart is empty");
    }

    const customer = await this.customerRepo.findById(customerId);

    // 2. Create order
    const order = Order.fromCart(cart, shippingAddress);

    // 3. Check inventory (call Inventory Context via OHS)
    const inventoryCheck = await this.inventoryService.checkAvailability(
      order.items
    );
    if (!inventoryCheck.available) {
      throw new Error(`Out of stock: ${inventoryCheck.unavailableItems}`);
    }

    // 4. Calculate pricing (call Pricing Context - Partnership)
    const pricing = await this.pricingService.calculateOrderPrice({
      items: order.items,
      customerId: customer.id,
      loyaltyTier: customer.loyaltyTier,
      shippingAddress: shippingAddress,
    });

    order.setPricing(pricing);

    // 5. Calculate shipping (call Shipping Context - Customer-Supplier)
    const shippingCost = await this.shippingService.calculateShipping({
      items: order.items,
      destination: shippingAddress,
    });

    // 6. Process payment (call Payment Context via ACL - Conformist)
    const paymentResult = await this.paymentService.processPayment({
      amount: pricing.total + shippingCost,
      paymentMethod: paymentMethod,
      orderId: order.id,
    });

    if (!paymentResult.success) {
      throw new Error("Payment failed: " + paymentResult.error);
    }

    // 7. Reserve inventory (call Inventory Context)
    await this.inventoryService.reserveStock(order.items);

    // 8. Confirm order and clear cart
    order.confirm();
    await this.orderRepo.save(order);
    cart.clear();
    await this.cartRepo.save(cart);

    // 9. Schedule shipment (publish event to Shipping Context)
    await this.shippingService.scheduleShipment({
      orderId: order.id,
      items: order.items,
      destination: shippingAddress,
    });

    return order;
  }
}
```

#### 2. Inventory Context - Supporting Subdomain

Manages stock across warehouses. Provides Open Host Service (REST API).

```javascript
// ===== INVENTORY CONTEXT (Bounded Context) =====

// This context's view of Product (Entity - different from Catalog!)
class InventoryProduct {
  constructor(productId, sku) {
    this.productId = productId;
    this.sku = sku;
    this.stockLevels = []; // Array of StockLevel per warehouse
  }

  getTotalAvailable() {
    return this.stockLevels.reduce((sum, level) => sum + level.available, 0);
  }

  reserve(quantity) {
    if (this.getTotalAvailable() < quantity) {
      throw new Error("Insufficient stock");
    }

    let remaining = quantity;
    for (const stockLevel of this.stockLevels) {
      if (remaining === 0) break;

      const toReserve = Math.min(remaining, stockLevel.available);
      stockLevel.reserve(toReserve);
      remaining -= toReserve;
    }
  }
}

class StockLevel {
  constructor(warehouseId, available, reserved) {
    this.warehouseId = warehouseId;
    this.available = available;
    this.reserved = reserved;
  }

  reserve(quantity) {
    if (quantity > this.available) {
      throw new Error("Insufficient available stock");
    }
    this.available -= quantity;
    this.reserved += quantity;
  }
}

// Domain Service exposed to other contexts (Open Host Service)
class InventoryService {
  constructor(productRepo) {
    this.productRepo = productRepo;
  }

  // Public API for other contexts
  async checkAvailability(orderItems) {
    const unavailableItems = [];

    for (const item of orderItems) {
      const product = await this.productRepo.findByProductId(item.productId);

      if (!product || product.getTotalAvailable() < item.quantity) {
        unavailableItems.push(item.productName);
      }
    }

    return {
      available: unavailableItems.length === 0,
      unavailableItems: unavailableItems,
    };
  }

  async reserveStock(orderItems) {
    for (const item of orderItems) {
      const product = await this.productRepo.findByProductId(item.productId);
      product.reserve(item.quantity);
      await this.productRepo.save(product);
    }
  }
}
```

#### 3. Pricing Context - Core Subdomain (Partnership with Sales)

Your competitive advantage: dynamic pricing, personalized discounts.

```javascript
// ===== PRICING CONTEXT (Bounded Context) =====

// Domain Service - Price calculation is complex domain logic
class PriceCalculator {
  constructor(discountRules, taxRates) {
    this.discountRules = discountRules;
    this.taxRates = taxRates;
  }

  calculate(pricingRequest) {
    // Complex business logic for dynamic pricing
    let subtotal = this.calculateSubtotal(pricingRequest.items);

    // Apply loyalty discount
    const loyaltyDiscount = this.calculateLoyaltyDiscount(
      subtotal,
      pricingRequest.loyaltyTier
    );
    subtotal -= loyaltyDiscount;

    // Apply promotional discounts (could be time-based, seasonal, etc.)
    const promoDiscount = this.applyPromotionalRules(
      pricingRequest.items,
      subtotal
    );
    subtotal -= promoDiscount;

    // Calculate tax based on shipping address
    const tax = this.calculateTax(
      subtotal,
      pricingRequest.shippingAddress.state
    );

    return {
      subtotal: subtotal,
      loyaltyDiscount: loyaltyDiscount,
      promotionalDiscount: promoDiscount,
      tax: tax,
      total: subtotal + tax,
    };
  }

  calculateSubtotal(items) {
    return items.reduce((sum, item) => {
      // Could use dynamic pricing here based on demand, time of day, etc.
      return sum + item.basePrice * item.quantity;
    }, 0);
  }

  calculateLoyaltyDiscount(amount, tier) {
    const discountRates = {
      GOLD: 0.15,
      SILVER: 0.1,
      BRONZE: 0.05,
    };
    return amount * (discountRates[tier] || 0);
  }

  applyPromotionalRules(items, subtotal) {
    // Example: Buy 2 get 10% off
    let discount = 0;
    // Complex promotional logic here...
    return discount;
  }

  calculateTax(amount, state) {
    return amount * (this.taxRates[state] || 0);
  }
}

// Service for Sales Context (Partnership relationship)
class PricingService {
  constructor(priceCalculator) {
    this.calculator = priceCalculator;
  }

  async calculateOrderPrice(pricingRequest) {
    return this.calculator.calculate(pricingRequest);
  }
}
```

#### 4. Shipping Context - Supporting Subdomain

```javascript
// ===== SHIPPING CONTEXT (Bounded Context) =====

// This context's view of address (Value Object)
class ShippingAddress {
  constructor(street, city, state, zipCode) {
    this.street = street;
    this.city = city;
    this.state = state;
    this.zipCode = zipCode;
    Object.freeze(this);
  }

  isRemote() {
    // Business logic for determining remote locations
    const remoteCodes = ["99501", "96701"]; // Alaska, Hawaii, etc.
    return remoteCodes.some((code) => this.zipCode.startsWith(code));
  }
}

class ShippableItem {
  constructor(productId, quantity, weight, dimensions) {
    this.productId = productId;
    this.quantity = quantity;
    this.weight = weight;
    this.dimensions = dimensions;
  }

  getTotalWeight() {
    return this.weight * this.quantity;
  }
}

// Domain Service exposed to other contexts
class ShippingService {
  async calculateShipping(request) {
    const address = new ShippingAddress(
      request.destination.street,
      request.destination.city,
      request.destination.state,
      request.destination.zipCode
    );

    // Convert from Sales Context items to Shipping Context items
    // This is where context translation happens
    const shippableItems = await this.translateToShippableItems(request.items);

    const totalWeight = shippableItems.reduce(
      (sum, item) => sum + item.getTotalWeight(),
      0
    );

    let cost = this.calculateBaseCost(totalWeight);

    if (address.isRemote()) {
      cost *= 1.5; // Remote surcharge
    }

    return cost;
  }

  async translateToShippableItems(orderItems) {
    // In reality, would fetch shipping data for each product
    return orderItems.map(
      (item) =>
        new ShippableItem(
          item.productId,
          item.quantity,
          2.5, // Example weight
          { length: 10, width: 8, height: 3 }
        )
    );
  }

  calculateBaseCost(weight) {
    if (weight < 1) return 5.99;
    if (weight < 5) return 9.99;
    if (weight < 10) return 14.99;
    return 19.99;
  }

  async scheduleShipment(shipmentRequest) {
    // Create shipment in this context
    const shipment = new Shipment(
      shipmentRequest.orderId,
      shipmentRequest.items,
      shipmentRequest.destination
    );
    // ... assign to carrier, print label, etc.
  }
}

class Shipment {
  constructor(orderId, items, destination) {
    this.id = this.generateShipmentId();
    this.orderId = orderId;
    this.items = items;
    this.destination = destination;
    this.status = "PENDING";
    this.carrier = null;
    this.trackingNumber = null;
  }

  generateShipmentId() {
    return "SHIP-" + Date.now();
  }
}
```

#### 5. Payment Context - Generic Subdomain with ACL

External payment system (Stripe). We use Anti-Corruption Layer to protect our domain.

```javascript
// ===== PAYMENT CONTEXT (External System with ACL) =====

// Anti-Corruption Layer - Protects Sales Context from Stripe's model
class PaymentServiceACL {
  constructor(stripeClient) {
    this.stripe = stripeClient;
  }

  async processPayment(paymentRequest) {
    try {
      // Translate from our domain to Stripe's API
      const stripeCharge = await this.stripe.charges.create({
        amount: this.toStripeCents(paymentRequest.amount),
        currency: "usd",
        source: paymentRequest.paymentMethod.token,
        description: `Order ${paymentRequest.orderId}`,
        metadata: {
          order_id: paymentRequest.orderId,
        },
      });

      // Translate Stripe's response back to our domain
      return this.translateStripeResponse(stripeCharge);
    } catch (error) {
      return {
        success: false,
        error: this.translateStripeError(error),
      };
    }
  }

  // Translation methods protect our domain from Stripe's model
  toStripeCents(dollars) {
    return Math.round(dollars * 100);
  }

  translateStripeResponse(stripeCharge) {
    return {
      success: stripeCharge.status === "succeeded",
      transactionId: stripeCharge.id,
      amount: stripeCharge.amount / 100,
      timestamp: new Date(stripeCharge.created * 1000),
    };
  }

  translateStripeError(error) {
    // Map Stripe error codes to our domain language
    const errorMap = {
      card_declined: "Payment method was declined",
      insufficient_funds: "Insufficient funds",
      expired_card: "Payment method has expired",
    };
    return errorMap[error.code] || "Payment processing error";
  }
}
```

### How It All Works Together

```javascript
// ===== USAGE: Checkout Flow Across Multiple Contexts =====

async function handleCheckoutRequest(req, res) {
  // Initialize services from different contexts
  const checkoutService = new CheckoutService(
    salesCartRepository,
    salesOrderRepository,
    salesCustomerRepository,
    new InventoryService(inventoryProductRepository), // Inventory Context
    new PricingService(priceCalculator), // Pricing Context
    new ShippingService(), // Shipping Context
    new PaymentServiceACL(stripeClient) // Payment Context (ACL)
  );

  try {
    const order = await checkoutService.checkout(
      req.body.customerId,
      req.body.shippingAddress,
      req.body.paymentMethod
    );

    res.json({
      success: true,
      orderId: order.id,
      total: order.pricing.total,
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message,
    });
  }
}
```

### What This Demonstrates

**Strategic Design:**

- ✅ **Multiple Bounded Contexts** - Sales, Inventory, Pricing, Shipping, Payment
- ✅ **Subdomain Classification** - Core (Sales, Pricing), Supporting (Inventory, Shipping), Generic (Payment)
- ✅ **Context Relationships** - Partnership, Customer-Supplier, Conformist, ACL
- ✅ **Clear Boundaries** - Each context has its own model of shared concepts (Customer, Product, etc.)

**Tactical Design:**

- ✅ **Entities** - Customer, Order, InventoryProduct, Shipment
- ✅ **Value Objects** - ShippingAddress
- ✅ **Aggregates** - ShoppingCart, Order with internal consistency
- ✅ **Repositories** - Separate repos per context
- ✅ **Domain Services** - PriceCalculator, ShippingService, InventoryService
- ✅ **Application Services** - CheckoutService orchestrates across contexts

**Integration Patterns:**

- ✅ **Open Host Service** - Inventory Context provides API
- ✅ **Anti-Corruption Layer** - Protects from Stripe's model
- ✅ **Translation** - Each context translates shared concepts to its own model
- ✅ **Context Mapping** - Explicit relationships between contexts

---

## 2.3.15 Key Insights from Strategic Design

1. **Same concept, different models**: "Product" means different things in Catalog vs Inventory vs Shipping

2. **Invest strategically**: Spend most effort on Core Subdomains (Pricing, Sales), less on Supporting (Shipping), minimal on Generic (Payment)

3. **Protect your domain**: Use ACL when integrating with external systems to prevent their model from polluting yours

4. **Make relationships explicit**: Context maps show dependencies and help coordinate teams

5. **Bounded Contexts enable autonomy**: Each context can evolve independently as long as the integration contracts remain stable

6. **Strategic design scales**: As the system grows, add new bounded contexts rather than making existing ones more complex

This strategic perspective is what separates DDD from just using tactical patterns. It's about organizing entire systems, not just individual classes.

---

## 2.3.16 Common Pitfalls and How to Avoid Them

DDD is powerful but can be misapplied. Here are the most common mistakes developers make when learning DDD.

### 2.3.16.1 The Anemic Domain Model

This is the most common DDD anti-pattern. An anemic domain model occurs when your entities contain almost no behavior—they're just data containers.

#### ❌ Anemic Model (BAD):

```javascript
// Just getters and setters - no business logic
class Order {
  constructor() {
    this.id = null;
    this.items = [];
    this.status = null;
    this.total = 0;
  }

  getId() {
    return this.id;
  }
  setId(id) {
    this.id = id;
  }
  getItems() {
    return this.items;
  }
  setItems(items) {
    this.items = items;
  }
  getStatus() {
    return this.status;
  }
  setStatus(status) {
    this.status = status;
  } // Allows any status!
  getTotal() {
    return this.total;
  }
  setTotal(total) {
    this.total = total;
  } // Can be set to anything!
}

// Business logic scattered in services
class OrderService {
  confirmOrder(order) {
    if (order.getItems().length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    order.setStatus("CONFIRMED");

    // Calculate total
    let total = 0;
    order.getItems().forEach((item) => {
      total += item.price * item.quantity;
    });
    order.setTotal(total);
  }
}
```

#### ✅ Rich Domain Model (GOOD):

```javascript
// Business logic lives in the entity
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = "PENDING";
  }

  addItem(product, quantity) {
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }
    this.items.push({ productId: product.id, price: product.price, quantity });
  }

  confirm() {
    if (this.items.length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    if (this.status !== "PENDING") {
      throw new Error("Order already confirmed");
    }
    this.status = "CONFIRMED";
  }

  getTotal() {
    return this.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );
  }
}
```

### 2.3.16.2 Over-Engineering

Don't apply all DDD patterns to every part of your application. Start simple and add complexity only when needed.

#### ❌ Over-Engineered (BAD):

```typescript
// For a simple user profile update - WAY too much!
interface UserRepository {
  findById(id: UserId): Promise<User>;
}

class UserId extends ValueObject<string> {}
class Email extends ValueObject<string> {}
class UserName extends ValueObject<string> {}

class User extends AggregateRoot {
  updateProfile(name: UserName, email: Email): void {
    this.apply(new UserProfileUpdatedEvent(this.id, name, email));
  }
}
```

#### ✅ Appropriate Complexity (GOOD):

```javascript
// Simple update - keep it simple
class User {
  constructor(id, name, email) {
    this.id = id;
    this.name = name;
    this.email = email;
  }

  updateProfile(name, email) {
    if (!this.isValidEmail(email)) {
      throw new Error("Invalid email");
    }
    this.name = name;
    this.email = email;
  }

  isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

### 2.3.16.3 Exposing Internal Aggregate State

Don't allow external code to directly modify aggregate internals.

#### ❌ Broken Encapsulation (BAD):

```javascript
class Order {
  constructor() {
    this.items = []; // Public array!
  }
}

// External code can bypass business rules
const order = await orderRepo.findById("123");
order.items.push({ productId: "abc", quantity: 5 }); // Bypasses validation!
order.status = "SHIPPED"; // Bypasses state transition rules!
```

#### ✅ Proper Encapsulation (GOOD):

```javascript
class Order {
  constructor() {
    this._items = []; // Private
    this._status = "PENDING";
  }

  // Controlled access through methods
  addItem(product, quantity) {
    if (this._status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }
    this._items.push({ productId: product.id, quantity });
  }

  // Read-only access
  getItems() {
    return [...this._items]; // Return copy, not reference
  }

  ship() {
    if (this._status !== "CONFIRMED") {
      throw new Error("Can only ship confirmed orders");
    }
    this._status = "SHIPPED";
  }
}
```

### 2.3.16.4 Creating One Universal Model

Trying to create one model that serves all bounded contexts.

#### ❌ Universal Model (BAD):

```javascript
// One giant Customer class for everything
class Customer {
  // Sales
  loyaltyTier;
  lifetimeValue;
  // Shipping
  shippingAddresses;
  // Support
  supportTickets;
  // Billing
  paymentMethods;
  // Marketing
  emailPreferences;
  // ... This is unmanageable!
}
```

#### ✅ Context-Specific Models (GOOD):

```javascript
// Sales Context
class Customer {
  loyaltyTier;
  lifetimeValue;
}

// Shipping Context
class Recipient {
  shippingAddresses;
  deliveryPreferences;
}

// Support Context
class SupportCustomer {
  supportTickets;
  vipStatus;
}
```

---

## 2.3.17 Key Takeaways

Let's summarize the fundamental principles of Domain-Driven Design:

**Core Principles:**

- **DDD puts business domain first:** Your code should reflect how the business actually works, not database structures or technical frameworks

- **Use Ubiquitous Language:** Developers and business people must speak the same language, and that language must appear in the code

**Tactical Design (Implementation):**

- **Entities have identity:** They can change over time but remain the same object because of their unique ID

- **Value Objects are immutable:** They're defined by their values and can't be changed—create new ones instead

- **Aggregates enforce consistency:** They group related objects and enforce all business rules through the aggregate root

- **Repositories hide storage details:** Your domain code doesn't need to know about databases or ORMs

- **Domain Services handle multi-object operations:** When behavior doesn't fit in one entity, use a domain service

**Strategic Design (Organization):**

- **Bounded Contexts define boundaries:** Each context has its own model and ubiquitous language

- **Subdomains guide investment:** Spend most effort on Core domains, less on Supporting, minimal on Generic

- **Context Maps make integration explicit:** Define how contexts relate using patterns like Partnership, ACL, OHS

- **Models can differ across contexts:** The same business concept can have different models in different contexts

**Best Practices:**

- **Rich models over anemic models:** Put business logic in your entities, not in service layers

- **Start simple, add complexity when needed:** Don't over-engineer. Apply DDD patterns where they add value

- **Both tactical and strategic are essential:** Tactical without strategic leads to a mess; strategic without tactical leads to poor implementation

- **DDD is for complex domains:** If your app is simple CRUD, you probably don't need DDD

## 2.3.18 Summary

Domain-Driven Design is fundamentally about aligning software with business reality at two levels:

**Tactical Design** provides patterns for implementing domain logic:

- Entities, Value Objects, and Aggregates model business concepts
- Repositories provide clean data access
- Domain Services handle multi-object operations
- Ubiquitous Language ensures code reflects business terms

**Strategic Design** provides patterns for organizing large systems:

- Bounded Contexts create clear model boundaries
- Subdomains help prioritize investment (Core, Supporting, Generic)
- Context Mapping makes integration explicit and managed

Together, they create software that:

- **Expresses business concepts clearly** in code
- **Enforces business rules** naturally
- **Remains flexible** as requirements change
- **Facilitates communication** between technical and business stakeholders
- **Scales** through clear boundaries and responsibilities

The key is knowing when to apply DDD: use it when business complexity justifies the investment, and keep things simple when it doesn't. Start with strategic design to understand the system landscape, then apply tactical patterns within each bounded context.

Remember: **the goal is not perfect code structure, but software that accurately models and supports the business domain.**
