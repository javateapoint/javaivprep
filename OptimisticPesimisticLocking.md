🚀 How Amazon Prevents Oversells on Black Friday: Optimistic vs Pessimistic Locking in Your Spring Boot Microservices 🚀

Picture this: it’s Black Friday at Amazon, and the last **Echo Dot** is flying off the virtual shelf.
Two customers click “Buy Now” at exactly the same millisecond.
Without the right locking strategy, both orders would succeed—and you’d end up with a negative inventory. 😱

How you can borrow Amazon’s playbook in your Spring Boot microservices to keep inventory accurate, customers happy ?

---

## 🕵️‍♂️ Real-World Amazon Scenario

- **Item**: Echo Dot (last unit)
- **Event**: Black Friday flash sale
- **Challenge**: Two buyers read `quantity=1` simultaneously  
- **Without Locking**: Both decrement → stock goes to -1 → angry customers + manual fixes

---

## 1. Optimistic Locking (“Trust & Verify”)

### When to Use
- Moderate contention (most SKUs aren’t hot-spots)
- You want non-blocking reads for maximum throughput
- You’re okay with a retry-on-conflict approach

### Amazon-Style Flow
1. Customer A & B both read `quantity=1`, `version=42`.  
2. A decrements → saves → DB row goes to `quantity=0`, `version=43`.  
3. B tries to save with `version=42` → conflict → throws `OptimisticLockException`.  
4. B retries: fresh read now sees `quantity=0` → “Out of Stock.”

### Code Snippet
```java
@Entity
@Table("inventory")
public class Inventory {
  @Id String sku;
  int quantity;
  @Version long version;    // ← JPA optimistic lock
  // …
}

@Service
public class InventoryService {
  @Transactional
  public void reserve(String sku, int qty) {
    int retries = 0;
    while (retries < 3) {
      try {
        Inventory inv = repo.findById(sku).orElseThrow();
        if (inv.getQuantity() < qty) throw new OutOfStockException(sku);
        inv.setQuantity(inv.getQuantity() - qty);
        repo.save(inv);       // includes version in WHERE clause
        return;
      } catch (OptimisticLockException e) {
        retries++;
        backOff(retries);     // exponential back-off
      }
    }
    throw new ConcurrencyFailureException(sku);
  }
}
```
✅ Pros: 
• Zero DB blocking 
• Scales for thousands of SKUs

⚠️ Cons: 
• Needs retry logic 
• Occasional rolled-back transactions





## 2. Pessimistic Locking (“Lock & Load”)

### When to Use
- **High contention** on a tiny set of items (e.g. “Last Echo Dot!”)  
- You need **strict one-at-a-time access**  
- You can **tolerate brief blocks**

---

### 📦 Amazon-Style Flow
1. **Thread A** executes `SELECT … FOR UPDATE` on the Echo Dot row → acquires lock  
2. **Thread B**’s attempt to read that row blocks until A commits/rolls back  
3. **Thread A** commits with `quantity=0` → lock released → **Thread B** proceeds and sees “Out of Stock.”

---

### 🔒 Code Snippet

```java
public interface InventoryRepo 
    extends JpaRepository<Inventory, String> {

  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("SELECT i FROM Inventory i WHERE i.sku = :sku")
  Inventory lockBySku(@Param("sku") String sku);
}

@Service
public class InventoryService {
  private final InventoryRepo repo;

  public InventoryService(InventoryRepo repo) {
    this.repo = repo;
  }

  @Transactional
  public void reserveWithLock(String sku, int qty) {
    Inventory inv = repo.lockBySku(sku);
    if (inv.getQuantity() < qty) {
      throw new OutOfStockException(sku);
    }
    inv.setQuantity(inv.getQuantity() - qty);
    // commit → lock released
  }
}
```

✅ Pros
- No version conflicts
- Straightforward business logic

⚠️ Cons
- Blocks concurrent threads
- Risk of DB deadlocks



| ⚖️ Aspect           | Optimistic (Amazon Batch) | Pessimistic (Last-Unit Sale) |
|---------------------|---------------------------|------------------------------|
| Contention Level    | Low → Moderate            | High                         |
| Throughput          | 🚀 High, non-blocking     | 🐢 Serial, blocking          |
| Complexity          | Requires retry logic      | Simpler code, DB locks       |
| Deadlock Risk       | None at DB level          | Possible                     |


