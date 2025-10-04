
# Practical @Transactional: Rollback Rules, Propagation, and Read-Only Patterns

**What this is:** a hands-on reference for `@Transactional` in Spring and Jakarta/JTA. It focuses on the behaviors you hit most: rollback rules (runtime vs checked), propagation (`REQUIRES_NEW`), and read-only queries—using simple, realistic “User/Order/Payment/Audit” flows.

**You’ll learn to:**

* Know when transactions **commit vs rollback** (and how to include checked exceptions).
* Use **`REQUIRES_NEW`** to commit work independently of the caller.
* Mark **read-only** methods for query paths.
* Decide between **Spring** `@Transactional` and **Jakarta** `@Transactional`.

**Assumptions:** Spring Data JPA repositories named `OrderRepo`, `PaymentRepo`, `AuditRepo` and corresponding entities. Replace names as needed; the patterns stay the same.

**Not in scope:** full project wiring, advanced isolation tuning, or multi-TM/XA setup.




```java







```






# @Transactional Examples with Sample Data`  

> Assumptions in snippets: you have `OrderRepo`, `PaymentRepo`, `AuditRepo` Spring Data JPA repositories and simple `Order`, `Payment`, `AuditLog` entities. Replace with your own names.

---

## A) Runtime exception → rolls back by default

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional
public void placeOrder_runtimeFails(Long userId, List<String> items, BigDecimal total) {
  Order order = new Order(userId, total, "CREATED");
  orderRepo.save(order);

  Payment payment = new Payment(order.getId(), total, "AUTHORIZED");
  paymentRepo.save(payment);

  // Runtime (unchecked) exception -> triggers rollback automatically
  throw new IllegalStateException("payment gateway down");
}
```

**Why:** Spring rolls back on unchecked exceptions (`RuntimeException`, `Error`) by default → both `Order` and `Payment` are undone.

---

## B) Checked exception → commits by default (surprising)

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional
public void placeOrder_checkedFails(Long userId, List<String> items, BigDecimal total) throws IOException {
  Order order = new Order(userId, total, "CREATED");
  orderRepo.save(order);

  Payment payment = new Payment(order.getId(), total, "AUTHORIZED");
  paymentRepo.save(payment);

  // Checked exception -> by default DOES NOT roll back
  throw new IOException("receipt printing failed");
}
```

**Why:** checked exceptions don’t trigger rollback unless you opt in → both rows commit.

---

## B-fixed) Include checked exceptions in rollback

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional(rollbackFor = Exception.class)
public void placeOrder_checkedFails_butRollback(Long userId, List<String> items, BigDecimal total) throws IOException {
  Order order = new Order(userId, total, "CREATED");
  orderRepo.save(order);

  Payment payment = new Payment(order.getId(), total, "AUTHORIZED");
  paymentRepo.save(payment);

  // Because rollbackFor=Exception.class, this checked exception triggers rollback
  throw new IOException("receipt printing failed");
}
```

**Why:** `rollbackFor = Exception.class` tells Spring to roll back on checked + unchecked → nothing persists.

---

## C) Outer fails; inner audit uses REQUIRES_NEW and survives

```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.annotation.Propagation;

@Transactional
public void placeOrder_withAudit_thenOuterFails(Long userId, List<String> items, BigDecimal total) {
  Order order = new Order(userId, total, "CREATED");
  orderRepo.save(order);

  Payment payment = new Payment(order.getId(), total, "AUTHORIZED");
  paymentRepo.save(payment);

  // Writes an audit entry in its own, independent transaction
  audit_requiresNew("Order " + order.getId() + " saved");

  // Outer fails afterward -> outer tx rolls back
  throw new RuntimeException("inventory mismatch");
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit_requiresNew(String message) {
  AuditLog log = new AuditLog(message);
  auditRepo.save(log);
}
```

**Why:** `audit_requiresNew` runs in its **own** transaction and commits. When the outer method throws, only the outer work (order/payment) rolls back; the audit row remains.

---

## D) Read-only query (no writes expected)

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional(readOnly = true)
public List<Order> listRecentOrders(int limit) {
  // Example finder; adjust to your repo method
  return orderRepo.findTopNByOrderByCreatedAtDesc(limit);
}
```

**Why:** `readOnly = true` is a hint/optimization for drivers/ORM; it doesn’t change semantics but communicates “no mutations here.”

---

## Optional: Jakarta version of “checked should roll back”

If you want the Jakarta style too, use its annotation on a method (note the different import):

```java
import jakarta.transaction.Transactional;

@Transactional(rollbackOn = Exception.class)
public void placeOrder_checkedFails_jakarta(Long userId, List<String> items, BigDecimal total) throws IOException {
  Order order = new Order(userId, total, "CREATED");
  orderRepo.save(order);

  Payment payment = new Payment(order.getId(), total, "AUTHORIZED");
  paymentRepo.save(payment);

  throw new IOException("receipt printing failed"); // will roll back due to rollbackOn
}
```

---

### Quick mapping to the effects you’ll see

* **Runtime error** → rollback (A).
* **Checked error** → commit (B) unless you add **`rollbackFor = Exception.class`** (B-fixed) or **`rollbackOn`** (Jakarta).
* **`REQUIRES_NEW`** → inner work commits even if outer later fails (C).
* **`readOnly = true`** → for queries (D).



```java







```









# Example Templates for `@Transactional` in Spring vs Jakarta/JTA


# 1) Basic default + “include checked exceptions”

```java
// --- Spring ---
import org.springframework.transaction.annotation.Transactional;

@Transactional // rollback on RuntimeException/Error by default
public void springDefault() { ... }

@Transactional(rollbackFor = Exception.class) // include checked exceptions
public void springChecked() throws Exception { ... }

// --- Jakarta/JTA ---
import jakarta.transaction.Transactional;

@Transactional // rollback on RuntimeException/Error by default
public void jtaDefault() { ... }

@Transactional(rollbackOn = Exception.class) // include checked exceptions
public void jtaChecked() throws Exception { ... }
```

# 2) Propagation: REQUIRED vs REQUIRES_NEW

```java
// --- Spring ---
@Transactional(propagation = Propagation.REQUIRED)
public void parentSpring() {
  ...;
  childSpring(); // joins same tx
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void childSpring() {
  ...; // suspends parent, new tx here
}

// --- Jakarta/JTA ---
@jakarta.transaction.Transactional // TxType.REQUIRED (default)
public void parentJta() {
  ...;
  childJta(); // joins same tx
}

@jakarta.transaction.Transactional(jakarta.transaction.Transactional.TxType.REQUIRES_NEW)
public void childJta() {
  ...; // runs in its own tx
}
```

# 3) Isolation + readOnly (Spring-only features)

```java
// --- Spring only ---
@Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true, timeout = 5)
public List<Item> loadItems() { ... }
```

> JTA/Jakarta `@Transactional` doesn’t have `isolation`, `readOnly`, or `timeout` attributes; you’d configure those at the datasource/transaction manager level.

# 4) NESTED (Spring-only) vs REQUIRED (approx in JTA)

```java
// --- Spring only ---
@Transactional
public void outer() {
  stepA();
  nestedPart(); // savepoint-based nested tx
  stepB();
}

@Transactional(propagation = Propagation.NESTED)
public void nestedPart() { ... }
```

> JTA has no `NESTED`. Closest behavior is manual savepoints in JDBC, or refactor to `REQUIRES_NEW` and accept different semantics.

# 5) Targeting a specific transaction manager (Spring)

Useful with multiple datasources.

```java
@Transactional(transactionManager = "ordersTxManager")
public void writeOrder() { ... }

@Transactional(transactionManager = "billingTxManager")
public void writeInvoice() { ... }
```

# 6) What happens with checked exceptions (the reason for `rollbackOn = Exception.class`)

```java
// --- Without rollbackFor/rollbackOn ---
@Transactional
public void springCheckedFail() throws IOException {
  repo.save(...);            // will COMMIT unless a RuntimeException/Error happens
  throw new IOException();   // checked => no rollback by default
}

// --- With rollbackFor/rollbackOn ---
@Transactional(rollbackFor = Exception.class)
public void springCheckedRollback() throws IOException {
  repo.save(...);
  throw new IOException();   // checked => triggers rollback
}

@jakarta.transaction.Transactional(rollbackOn = Exception.class)
public void jtaCheckedRollback() throws Exception { ... }
```

# 7) Catching exceptions (and still rolling back)

If you catch the exception, the framework can’t see it—so it will commit unless you mark rollback-only.

```java
// --- Spring: mark rollback only ---
import org.springframework.transaction.interceptor.TransactionAspectSupport;

@Transactional
public void catchButRollbackSpring() {
  try {
    doWork();
  } catch (Exception ex) {
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    // optionally rethrow, or swallow if you must
  }
}

// --- Jakarta/JTA: mark rollback only ---
import jakarta.transaction.TransactionSynchronizationRegistry;
import jakarta.annotation.Resource;

public class Service {
  @Resource
  private TransactionSynchronizationRegistry tsr;

  @jakarta.transaction.Transactional
  public void catchButRollbackJta() {
    try {
      doWork();
    } catch (Exception ex) {
      tsr.setRollbackOnly(); // mark current tx for rollback
    }
  }
}
```

# 8) Self-invocation pitfall (applies to both; especially Spring proxies)

```java
@Service
public class MyService {
  @Transactional
  public void outer() {
    // self-invocation: this.inner() bypasses the proxy -> NO new tx attributes applied
    inner();
  }

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void inner() { ... }
}
```

**Fixes:** call `inner()` from another bean, or inject self via interface/proxy and call through that, or use AspectJ weaving.

# 9) Quick checklist

* Pick **one** annotation style project-wide (Spring vs Jakarta) to avoid confusion.
* Need `isolation`, `readOnly`, `timeout`, `NESTED`, or per-manager control? → **Spring**.
* Want portable, minimal config? → **Jakarta**.
* Throwing checked exceptions and still want rollback? → `rollbackFor = Exception.class` (Spring) or `rollbackOn = Exception.class` (Jakarta).
* If you catch exceptions, **mark rollback-only** yourself (or rethrow).
