# 2.4 Command Query Responsibility Segregation (CQRS)

## 2.4.1 What Is CQRS?

**Command Query Responsibility Segregation (CQRS)** is a design pattern that separates read operations (queries) from write operations (commands) into distinct models. Instead of using one model for both reading and writing data, CQRS uses different models optimized for each purpose.

The core principle is simple:

> **Separate the model that changes data from the model that reads data.**

### CQRS vs Traditional CRUD

In traditional CRUD (Create, Read, Update, Delete) applications, we use the same model for everything:

```javascript
// Traditional CRUD - Same model for reads and writes
class Order {
  constructor(id, customerId, items, status, total) {
    this.id = id;
    this.customerId = customerId;
    this.items = items;
    this.status = status;
    this.total = total;
    this.createdAt = new Date();
    this.updatedAt = new Date();
  }

  // Used for both reading and writing
  save() {
    /* update database */
  }
  static findById(id) {
    /* read from database */
  }
  static findByCustomer(customerId) {
    /* read from database */
  }
}
```

With CQRS, we split this into two separate models:

```javascript
// CQRS - Separate models

// WRITE MODEL (Command side)
class OrderCommand {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = "PENDING";
  }

  addItem(product, quantity) {
    // Complex business logic
    if (this.status !== "PENDING") {
      throw new Error("Cannot modify confirmed order");
    }
    this.items.push({ productId: product.id, quantity });
  }

  confirm() {
    // Validation and state changes
    if (this.items.length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    this.status = "CONFIRMED";
  }
}

// READ MODEL (Query side)
class OrderQuery {
  // Simple data transfer object - no business logic
  constructor(data) {
    this.id = data.id;
    this.customerName = data.customerName;
    this.itemCount = data.items.length;
    this.total = data.total;
    this.status = data.status;
    this.createdAt = data.createdAt;
  }

  // Just returns data, no modifications
}
```

---

## 2.4.2 Why Does CQRS Exist?

As applications grow, using one model for both reads and writes creates problems.

### The Problem with Traditional CRUD

A traditional e-commerce with the CRUD approach would look like this:

```javascript
class Order {
  constructor() {
    this.id = null;
    this.customerId = null;
    this.customerName = null;
    this.customerEmail = null;
    this.shippingAddress = null;
    this.billingAddress = null;
    this.items = [];
    this.subtotal = 0;
    this.tax = 0;
    this.shipping = 0;
    this.total = 0;
    this.status = null;
    this.createdAt = null;
    this.updatedAt = null;
    this.confirmedAt = null;
    this.shippedAt = null;
    // ... many more fields
  }

  // This method tries to do too much
  async save() {
    // Complex validation
    // Complex business rules
    // Database updates
    // Possible joins across multiple tables
  }

  // Reading is complicated too
  static async findById(id) {
    // Needs to join multiple tables
    // Loads ALL fields even if you only need a few
    // Slow because it's not optimized for reading
  }
}
```

**Problems with this approach:**

1. **Performance Issues**

   - Reading order details requires joining multiple tables
   - Writing an order update locks rows that readers might need
   - Can't optimize database indexes for both reads and writes

2. **Data Mismatch**

   - When displaying order history, you don't need shipping address
   - When confirming an order, you don't need customer email
   - But the model includes everything for every operation

3. **Lock Contention**

   - Multiple users reading orders can block users trying to update orders
   - Database locks affect both operations equally

4. **Security Challenges**
   - Read operations might expose fields that should only be used for writes
   - Hard to apply different security rules for reading vs writing

### The CQRS Solution

CQRS solves these problems by creating two separate models:

```javascript
// ===== WRITE MODEL (Commands) =====
// Focused on business logic and data changes

class CreateOrderCommand {
  constructor(customerId, items, shippingAddress) {
    this.customerId = customerId;
    this.items = items;
    this.shippingAddress = shippingAddress;
  }
}

class OrderCommandHandler {
  async handle(command) {
    // Validate
    if (!command.items || command.items.length === 0) {
      throw new Error("Order must have items");
    }

    // Apply business logic
    const order = new Order(command.customerId);
    command.items.forEach((item) => {
      order.addItem(item.productId, item.quantity);
    });

    // Save to database (optimized for writes)
    await this.orderRepository.save(order);

    return order.id;
  }
}

// ===== READ MODEL (Queries) =====
// Focused on fast data retrieval

class OrderListQuery {
  constructor(customerId) {
    this.customerId = customerId;
  }
}

class OrderListQueryHandler {
  async handle(query) {
    // Optimized read - might use different database
    // or denormalized table for fast retrieval
    const result = await this.database.query(
      `
      SELECT 
        o.id,
        o.total,
        o.status,
        o.created_at,
        COUNT(oi.id) as item_count
      FROM orders o
      LEFT JOIN order_items oi ON o.id = oi.order_id
      WHERE o.customer_id = ?
      GROUP BY o.id
      ORDER BY o.created_at DESC
    `,
      [query.customerId]
    );

    return result.map((row) => ({
      id: row.id,
      total: row.total,
      status: row.status,
      itemCount: row.item_count,
      createdAt: row.created_at,
    }));
  }
}
```

---

## 2.4.3 When Should You Use CQRS?

CQRS is not always the right choice.

### ✅ Use CQRS When:

**1. Read and Write Patterns Differ Significantly**

```javascript
// Example: E-commerce product catalog

// WRITES are infrequent:
// - Add new product (once)
// - Update price (occasionally)
// - Update stock (after each sale)

// READS are constant:
// - Browse catalog (thousands per minute)
// - Search products (hundreds per minute)
// - View product details (thousands per minute)

// CQRS lets you optimize each side independently
```

**2. Complex Domain Logic on Writes**

```javascript
// Example: Order processing with complex validation

class PlaceOrderCommand {
  async execute(order) {
    // Complex business rules:
    // - Validate customer credit limit
    // - Check inventory availability
    // - Calculate complex pricing with discounts
    // - Apply business rules (can't order on hold, etc.)
    // - Allocate inventory
    // - Create payment authorization
    // Read model doesn't need any of this complexity
  }
}
```

**3. Different Teams Work on Reads vs Writes**

- Backend team focuses on complex order processing (writes)
- Frontend team focuses on displaying data (reads)
- Teams can work independently

**4. Performance Optimization is Critical**

```javascript
// Example: Dashboard with aggregated statistics

// WRITE MODEL: Simple order creation
await orderCommandHandler.createOrder(orderData);

// READ MODEL: Pre-calculated dashboard data
const dashboard = await dashboardQueryHandler.getDashboard();
// Returns pre-aggregated data:
// - Total sales today (calculated in background)
// - Top selling products (materialized view)
// - Customer statistics (denormalized data)
```

**Examples where CQRS shines:**

- E-commerce platforms
- Financial trading systems
- Reporting and analytics dashboards
- Collaborative applications (Google Docs-style)
- Task management systems

### ❌ Avoid CQRS When:

**1. Simple CRUD Operations**

```javascript
// Example: User profile management
// Reads and writes are equally simple
class UserProfile {
  updateName(name) {
    this.name = name;
  }
  getName() {
    return this.name;
  }
}
// CQRS adds unnecessary complexity here
```

**2. Business Logic is Minimal**

If your application is mostly data entry and display, CQRS is overkill.

**3. Reads and Writes Have Similar Complexity**

If both operations are equally complex, you won't gain much from separation.

**4. Small Applications**

Don't use CQRS just because it's trendy. Start simple and add CQRS only when you actually need it.

**Examples where CQRS is overkill:**

- Simple blogs
- Basic contact forms
- Small internal tools
- CRUD admin panels

---

## 2.4.4 CQRS Fundamentals

### 2.4.4.1 Commands: Changing State

**Commands** represent intentions to change the system state. They are imperative - they tell the system to do something.

#### Characteristics of Commands:

- **Task-based**: Represent business operations, not just data changes
- **Imperative**: Named as actions (PlaceOrder, CancelSubscription)
- **Can fail**: Should include validation and business rules
- **Return minimal data**: Usually just success/failure and maybe an ID

#### Example: Order Commands

```javascript
// ===== COMMANDS =====

// Command 1: Create a new order
class CreateOrderCommand {
  constructor(customerId, items, shippingAddress, paymentMethod) {
    this.customerId = customerId;
    this.items = items;
    this.shippingAddress = shippingAddress;
    this.paymentMethod = paymentMethod;
  }
}

// Command 2: Cancel an existing order
class CancelOrderCommand {
  constructor(orderId, reason) {
    this.orderId = orderId;
    this.reason = reason;
  }
}

// Command 3: Add item to order
class AddItemToOrderCommand {
  constructor(orderId, productId, quantity) {
    this.orderId = orderId;
    this.productId = productId;
    this.quantity = quantity;
  }
}

// ===== COMMAND HANDLERS =====
// Handle the actual business logic

class OrderCommandHandler {
  constructor(orderRepository, inventoryService, paymentService) {
    this.orderRepository = orderRepository;
    this.inventoryService = inventoryService;
    this.paymentService = paymentService;
  }

  async handleCreateOrder(command) {
    // 1. Validate command
    if (!command.items || command.items.length === 0) {
      throw new Error("Order must contain items");
    }

    // 2. Check inventory
    for (const item of command.items) {
      const available = await this.inventoryService.checkStock(
        item.productId,
        item.quantity
      );
      if (!available) {
        throw new Error(`Product ${item.productId} out of stock`);
      }
    }

    // 3. Create order (domain logic)
    const order = new Order(command.customerId);
    command.items.forEach((item) => {
      order.addItem(item.productId, item.quantity, item.price);
    });
    order.setShippingAddress(command.shippingAddress);

    // 4. Process payment
    const paymentResult = await this.paymentService.authorize(
      command.paymentMethod,
      order.getTotal()
    );

    if (!paymentResult.success) {
      throw new Error("Payment authorization failed");
    }

    // 5. Save order
    order.confirm();
    await this.orderRepository.save(order);

    // 6. Return minimal result
    return {
      success: true,
      orderId: order.id,
    };
  }

  async handleCancelOrder(command) {
    // 1. Load order
    const order = await this.orderRepository.findById(command.orderId);

    if (!order) {
      throw new Error("Order not found");
    }

    // 2. Apply business rule
    if (order.status === "SHIPPED") {
      throw new Error("Cannot cancel shipped order");
    }

    // 3. Update state
    order.cancel(command.reason);

    // 4. Save
    await this.orderRepository.save(order);

    return { success: true };
  }

  async handleAddItemToOrder(command) {
    // 1. Load order
    const order = await this.orderRepository.findById(command.orderId);

    // 2. Validate
    if (order.status !== "PENDING") {
      throw new Error("Can only add items to pending orders");
    }

    // 3. Check inventory
    const available = await this.inventoryService.checkStock(
      command.productId,
      command.quantity
    );
    if (!available) {
      throw new Error("Product out of stock");
    }

    // 4. Add item
    order.addItem(command.productId, command.quantity);

    // 5. Save
    await this.orderRepository.save(order);

    return { success: true };
  }
}
```

**Key Points:**

- Commands are objects, not just method calls
- Each command has a dedicated handler
- Handlers contain business logic and validation
- Commands can be queued for asynchronous processing

---

### 2.4.4.2 Queries: Reading Data

**Queries** retrieve data without changing system state. They are passive - they just read and return.

#### Characteristics of Queries:

- **Read-only**: Never modify data
- **Return DTOs**: Data Transfer Objects, not domain entities
- **Optimized for display**: Structured for UI needs
- **No business logic**: Just data retrieval and transformation

#### Example: Order Queries

```javascript
// ===== QUERIES =====

// Query 1: Get order details for display
class GetOrderDetailsQuery {
  constructor(orderId) {
    this.orderId = orderId;
  }
}

// Query 2: List customer's orders
class GetCustomerOrdersQuery {
  constructor(customerId, pageNumber, pageSize) {
    this.customerId = customerId;
    this.pageNumber = pageNumber;
    this.pageSize = pageSize;
  }
}

// Query 3: Get order statistics
class GetOrderStatisticsQuery {
  constructor(startDate, endDate) {
    this.startDate = startDate;
    this.endDate = endDate;
  }
}

// ===== DATA TRANSFER OBJECTS (DTOs) =====
// Simple objects for returning data

class OrderDetailsDTO {
  constructor(data) {
    this.id = data.id;
    this.orderNumber = data.orderNumber;
    this.customerName = data.customerName;
    this.status = data.status;
    this.items = data.items.map((item) => ({
      productName: item.productName,
      quantity: item.quantity,
      unitPrice: item.unitPrice,
      subtotal: item.quantity * item.unitPrice,
    }));
    this.subtotal = data.subtotal;
    this.tax = data.tax;
    this.shipping = data.shipping;
    this.total = data.total;
    this.createdAt = data.createdAt;
  }
}

class OrderListItemDTO {
  constructor(data) {
    this.id = data.id;
    this.orderNumber = data.orderNumber;
    this.status = data.status;
    this.total = data.total;
    this.itemCount = data.itemCount;
    this.createdAt = data.createdAt;
  }
}

// ===== QUERY HANDLERS =====
// Retrieve and transform data

class OrderQueryHandler {
  constructor(database) {
    this.db = database;
  }

  async handleGetOrderDetails(query) {
    // Optimized query - might use denormalized view
    const result = await this.db.query(
      `
      SELECT 
        o.id,
        o.order_number,
        c.name as customer_name,
        o.status,
        o.subtotal,
        o.tax,
        o.shipping,
        o.total,
        o.created_at,
        JSON_ARRAYAGG(
          JSON_OBJECT(
            'productName', p.name,
            'quantity', oi.quantity,
            'unitPrice', oi.unit_price
          )
        ) as items
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      JOIN order_items oi ON o.id = oi.order_id
      JOIN products p ON oi.product_id = p.id
      WHERE o.id = ?
      GROUP BY o.id
    `,
      [query.orderId]
    );

    if (!result || result.length === 0) {
      return null;
    }

    return new OrderDetailsDTO(result[0]);
  }

  async handleGetCustomerOrders(query) {
    const offset = (query.pageNumber - 1) * query.pageSize;

    // Simple, fast query optimized for listing
    const results = await this.db.query(
      `
      SELECT 
        o.id,
        o.order_number,
        o.status,
        o.total,
        o.created_at,
        COUNT(oi.id) as item_count
      FROM orders o
      LEFT JOIN order_items oi ON o.id = oi.order_id
      WHERE o.customer_id = ?
      GROUP BY o.id
      ORDER BY o.created_at DESC
      LIMIT ? OFFSET ?
    `,
      [query.customerId, query.pageSize, offset]
    );

    return results.map((row) => new OrderListItemDTO(row));
  }

  async handleGetOrderStatistics(query) {
    // Aggregated query for dashboard
    const result = await this.db.query(
      `
      SELECT 
        COUNT(*) as total_orders,
        SUM(total) as total_revenue,
        AVG(total) as average_order_value,
        COUNT(DISTINCT customer_id) as unique_customers
      FROM orders
      WHERE created_at BETWEEN ? AND ?
    `,
      [query.startDate, query.endDate]
    );

    return {
      totalOrders: result[0].total_orders,
      totalRevenue: result[0].total_revenue,
      averageOrderValue: result[0].average_order_value,
      uniqueCustomers: result[0].unique_customers,
    };
  }
}
```

**Key Points:**

- Queries are objects that describe what data you want
- Query handlers fetch and transform the data
- Returns DTOs, not domain entities
- Can use different database optimizations (indexes, views, caching)

---

## 2.4.5 CQRS Implementations

There are different levels of CQRS implementation, from simple to complex.

### 2.4.5.1 Basic CQRS: Same Database, Different Models

The simplest CQRS implementation uses the same database but separates the code for reads and writes.

```
┌──────────────────────────────────────────────┐
│            APPLICATION LAYER                 │
├──────────────────────────────────────────────┤
│                                              │
│  Commands              Queries              │
│  ┌────────────┐       ┌────────────┐       │
│  │ CreateOrder│       │ GetOrders  │       │
│  │ CancelOrder│       │ GetDetails │       │
│  └─────┬──────┘       └─────┬──────┘       │
│        │                    │               │
│        ▼                    ▼               │
│  ┌────────────┐       ┌────────────┐       │
│  │  Command   │       │   Query    │       │
│  │  Handlers  │       │  Handlers  │       │
│  └─────┬──────┘       └─────┬──────┘       │
│        │                    │               │
└────────┼────────────────────┼───────────────┘
         │                    │
         │    ┌───────────────┤
         │    │               │
         ▼    ▼               ▼
    ┌────────────────────────────┐
    │     SAME DATABASE           │
    │                            │
    │  Tables optimized with:    │
    │  - Indexes for writes      │
    │  - Indexes for reads       │
    │  - Possibly denormalized   │
    │    tables for common reads │
    └────────────────────────────┘
```

#### Example Implementation:

```javascript
// ===== WRITE MODEL =====
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = "PENDING";
  }

  addItem(productId, quantity, price) {
    this.items.push({ productId, quantity, price });
  }

  confirm() {
    if (this.items.length === 0) {
      throw new Error("Cannot confirm empty order");
    }
    this.status = "CONFIRMED";
  }
}

class OrderWriteRepository {
  async save(order) {
    // Save to orders table (normalized)
    await this.db.query(
      `
      INSERT INTO orders (id, customer_id, status, created_at)
      VALUES (?, ?, ?, ?)
      ON DUPLICATE KEY UPDATE status = ?
    `,
      [order.id, order.customerId, order.status, new Date(), order.status]
    );

    // Save items to order_items table
    for (const item of order.items) {
      await this.db.query(
        `
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (?, ?, ?, ?)
      `,
        [order.id, item.productId, item.quantity, item.price]
      );
    }
  }
}

// ===== READ MODEL =====
class OrderReadRepository {
  async findOrderSummary(orderId) {
    // Use denormalized view for fast reads
    const result = await this.db.query(
      `
      SELECT 
        o.id,
        o.order_number,
        c.name as customer_name,
        o.status,
        o.total,
        o.created_at
      FROM order_summary_view osv  -- Denormalized view
      WHERE osv.id = ?
    `,
      [orderId]
    );

    return result[0];
  }

  async findCustomerOrders(customerId) {
    // Optimized read query
    const results = await this.db.query(
      `
      SELECT * FROM order_list_view
      WHERE customer_id = ?
      ORDER BY created_at DESC
    `,
      [customerId]
    );

    return results;
  }
}
```

**Benefits:**

- Simpler than separate databases
- Can optimize queries independently from writes
- Clear separation of concerns in code
- Easier to understand and maintain than full CQRS

**When to use:**

- Starting with CQRS
- Don't need separate database scaling
- Want cleaner code separation

---

### 2.4.5.2 Advanced CQRS: Separate Databases

For high-scale systems, you can use completely separate databases for reads and writes.

```
┌──────────────────────────────────────────────┐
│            APPLICATION LAYER                 │
├──────────────────────────────────────────────┤
│                                              │
│  Commands              Queries              │
│  ┌────────────┐       ┌────────────┐       │
│  │ CreateOrder│       │ GetOrders  │       │
│  │ CancelOrder│       │ GetDetails │       │
│  └─────┬──────┘       └─────┬──────┘       │
│        │                    │               │
│        ▼                    ▼               │
│  ┌────────────┐       ┌────────────┐       │
│  │  Command   │       │   Query    │       │
│  │  Handlers  │       │  Handlers  │       │
│  └─────┬──────┘       └─────┬──────┘       │
│        │                    │               │
└────────┼────────────────────┼───────────────┘
         │                    │
         │                    │
         ▼                    ▼
    ┌──────────┐        ┌──────────┐
    │  WRITE   │        │   READ   │
    │ DATABASE │        │ DATABASE │
    │          │        │          │
    │ (SQL)    │        │ (NoSQL   │
    │ Normalized│       │  or      │
    │          │        │ Denorm.) │
    └────┬─────┘        └────▲─────┘
         │                   │
         │  Synchronization  │
         └───────────────────┘
           (Events/Messages)
```

#### Example Implementation:

```javascript
// ===== WRITE SIDE =====

class OrderCommandHandler {
  constructor(writeDb, eventBus) {
    this.writeDb = writeDb; // SQL database
    this.eventBus = eventBus;
  }

  async createOrder(command) {
    // 1. Create order in write database (normalized)
    const order = new Order(command.customerId);
    command.items.forEach((item) => {
      order.addItem(item.productId, item.quantity, item.price);
    });
    order.confirm();

    await this.writeDb.saveOrder(order);

    // 2. Publish event for read side
    await this.eventBus.publish({
      type: "OrderCreated",
      data: {
        orderId: order.id,
        customerId: order.customerId,
        items: order.items,
        total: order.getTotal(),
        createdAt: new Date(),
      },
    });

    return { orderId: order.id };
  }
}

// ===== READ SIDE =====

class OrderEventHandler {
  constructor(readDb) {
    this.readDb = readDb; // Could be MongoDB, Redis, etc.
  }

  // Listen to events from write side
  async onOrderCreated(event) {
    // Update read database with denormalized data
    await this.readDb.saveOrderSummary({
      id: event.data.orderId,
      customerId: event.data.customerId,
      itemCount: event.data.items.length,
      total: event.data.total,
      status: "CONFIRMED",
      createdAt: event.data.createdAt,
      // Denormalized data for fast reads
      items: event.data.items.map((item) => ({
        productId: item.productId,
        productName: item.productName, // Pre-fetched
        quantity: item.quantity,
        price: item.price,
      })),
    });
  }

  async onOrderCancelled(event) {
    await this.readDb.updateOrderStatus(event.data.orderId, "CANCELLED");
  }
}

class OrderQueryHandler {
  constructor(readDb) {
    this.readDb = readDb; // NoSQL or denormalized SQL
  }

  async getOrderDetails(query) {
    // Fast read from denormalized database
    const order = await this.readDb.findOrderSummary(query.orderId);

    return {
      id: order.id,
      itemCount: order.itemCount,
      total: order.total,
      status: order.status,
      items: order.items, // Already includes product names
    };
  }
}
```

**Benefits:**

- **Independent scaling**: Scale read database separately from write database
- **Technology choice**: Use SQL for writes, NoSQL for reads
- **Performance**: Optimize each database for its purpose
- **Resilience**: Read database failure doesn't affect writes

**Challenges:**

- **Eventual consistency**: Read database might lag behind writes
- **Complexity**: Need to synchronize two databases
- **Infrastructure cost**: Running two databases

**When to use:**

- High read/write ratio (100:1 or more)
- Need to scale reads and writes independently
- Can tolerate eventual consistency

---

## 2.4.8 Common Pitfalls and How to Avoid Them

### 2.4.8.1 Eventual Consistency Issues

**Problem:** When using separate databases, the read database might lag behind writes.

```javascript
// User creates an order
await orderCommandService.createOrder(command);

// Immediately tries to view it
const order = await orderQueryService.getOrderDetails(orderId);
// Might return null because read database hasn't updated yet!
```

**Solution:** Handle eventual consistency in the UI

```javascript
// Option 1: Return projected data from command
async createOrder(command) {
  const order = await this.handleCreateOrder(command);

  // Return data that UI can display immediately
  return {
    orderId: order.id,
    status: 'PENDING',
    message: 'Order created successfully'
  };
}

// Option 2: Poll for data
async function waitForOrder(orderId, maxAttempts = 5) {
  for (let i = 0; i < maxAttempts; i++) {
    const order = await orderQueryService.getOrderDetails(orderId);
    if (order) return order;
    await sleep(200); // Wait 200ms
  }
  throw new Error('Order not found');
}

// Option 3: WebSocket update when read model updates
orderEventHandler.on('OrderCreated', (event) => {
  websocket.send({ type: 'OrderReady', orderId: event.orderId });
});
```

### 2.4.8.2 Over-Complicating Simple Operations

**Problem:** Applying CQRS to everything, even simple CRUD.

```javascript
// ❌ BAD: CQRS for simple user profile
class UpdateUserNameCommand {}
class GetUserNameQuery {}
// Too much ceremony for "user.name = newName"
```

**Solution:** Use CQRS selectively

```javascript
// ✓ GOOD: Simple updates don't need CQRS
class UserProfile {
  updateName(name) {
    this.name = name;
  }
}

// ✓ GOOD: Complex operations benefit from CQRS
class PlaceOrderCommand {} // Has validation, inventory, payment
class GetOrderHistoryQuery {} // Optimized read
```

### 2.4.8.3 Not Optimizing Read Models

**Problem:** Read model is just as complex as write model.

```javascript
// ❌ BAD: Read model with lots of joins
async getOrderDetails(orderId) {
  const order = await db.query(`
    SELECT * FROM orders
    JOIN order_items ON ...
    JOIN products ON ...
    JOIN customers ON ...
    JOIN addresses ON ...
    -- Too complex!
  `);
}
```

**Solution:** Create denormalized views

```javascript
// ✓ GOOD: Pre-calculated view
CREATE VIEW order_details_view AS
SELECT
  o.id,
  o.order_number,
  c.name as customer_name,
  o.total,
  -- All data pre-joined
FROM orders o
JOIN customers c ON o.customer_id = c.id;

// Simple, fast query
async getOrderDetails(orderId) {
  return await db.queryOne(
    'SELECT * FROM order_details_view WHERE id = ?',
    [orderId]
  );
}
```

---

## 2.4.9 Key Takeaways

**Core Principles:**

- **CQRS separates reads from writes:** Different models optimized for different purposes
- **Commands change state:** Imperative, task-based, can fail
- **Queries read state:** Return DTOs, never modify data, optimized for display

**When to Use CQRS:**

- ✅ Read/write patterns differ significantly
- ✅ Complex business logic on write side
- ✅ Performance optimization is critical
- ✅ Different teams handle reads vs writes
- ❌ Simple CRUD operations
- ❌ Minimal business logic
- ❌ Small applications

**Implementation Levels:**

1. **Basic CQRS:** Same database, separate code models
2. **Advanced CQRS:** Separate databases with synchronization
3. **CQRS + Event Sourcing:** Event log as write store

**Benefits:**

- Independent scaling of reads and writes
- Optimized data schemas for each purpose
- Better security separation
- Cleaner, more maintainable code
- Simpler queries on read side

**Challenges:**

- Increased complexity
- Eventual consistency (with separate databases)
- Need for synchronization mechanisms
- More infrastructure to manage
