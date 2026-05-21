# 🎯 🟢 Day 9 of Beginner (Level 1 of 7): Place a Simple Order

**Overall Day**: Day 9 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 9 of 20
**Today's Theme**: Cart ko ek confirmed **order** mein convert karna — jahan order banana, order-items save karna, stock kam karna, aur cart khaali karna **sab ek saath ya bilkul nahi** (atomic transaction) hona chahiye; double-click pe do order na banein (idempotency); aur total **server pe** recompute ho (client pe trust nahi).

---

## 📖 The Brainer Scenario (Real Problem)

Bhai, ab tak ka safar yaad karo: Day 7 mein humne item cart mein **daala**, Day 8 mein cart ka **total** paisa-perfect nikala. Aaj woh decisive moment hai — user **"Place Order"** button dabata hai. Ab ek browsing session ko ek **legally binding order** ban jana hai.

Dekhne mein simple lagta hai: `INSERT INTO orders`, kaam khatam. Lekin yeh wahi jagah hai jahan e-commerce ki asli engineering chhupi hai. Ek single "Place Order" click ke peeche **chaar cheezein ek saath honi chahiye**:

```
"Place Order" click
        │
        ├─▶ 1. orders table mein naya order banao (status = PLACED)
        ├─▶ 2. har cart item ko order_items mein copy karo (price LOCK karke)
        ├─▶ 3. products table mein stock kam karo (qty minus)
        └─▶ 4. user ka cart khaali karo
```

Ab socho: agar step 3 ke baad server crash ho gaya? Order ban gaya, stock kam ho gaya, lekin cart clear nahi hua — ya us se bhi bura, order ban gaya lekin stock kam nahi hua aur tumne ek aisi cheez bech di jo stock mein thi hi nahi. Yeh ko **partial failure** kehte hain, aur yeh production ka sabse khatarnaak demon hai.

**The Real Challenge (The "Gotcha")**:

1. **Atomicity — sab ya kuch nahi.** Yeh chaar steps ek **transaction** mein hone chahiye. Transaction = database ka woh waada: *"Yeh saare operations ya to sab ek saath commit honge, ya koi bhi nahi (rollback).* Beech mein crash hua to DB sab undo kar deta hai. Bina iske tumhara data **inconsistent** ho jayega — order hai par stock galat, ya paisa kata par order gayab.

2. **Double-submit / Idempotency.** User ne "Place Order" dabaya, internet slow tha, response aaya nahi, usne **dobara** dabaya. Ab? Do orders ban gaye, do baar stock kata, do baar paisa! **Idempotency** ka matlab: *"Same request chahe 1 baar bhejo ya 10 baar, result same rahega — sirf ek hi order banega."* (Idem = same, potent = power → "same effect har baar".)

3. **Server-side recompute (security).** Day 8 ka sabaq: client jo total bhejta hai uspe **bharosa nahi**. Place order pe backend cart ko **dobara** uthata hai, live price + live stock + coupon validity check karke total **khud** banata hai. Warna user DevTools se `total: 1` bhej ke iPhone Rs 1 mein le jayega.

4. **Stock oversell.** 100 log ek saath aakhri 1 iPhone khareedne ki koshish karein. Bina proper locking ke, sab ka stock-check "haan 1 available hai" dikhayega, sab order place kar denge — aur tumne **1 phone 100 logon ko** bech diya. (Yeh concurrency wala demon kal-parson detail mein dekhenge; aaj basic safe version.)

5. **Aaj ka OOP lesson (Method Overloading vs Overriding)**: `placeOrder()` ke kai roop — koi coupon ke saath, koi guest checkout, koi saved address ke saath. Yeh **overloading** (compile-time, same naam alag parameters) ka case hai. Aur order ke baad notification — Email/SMS/WhatsApp — yeh **overriding** (run-time, child class parent ka behavior badalta hai) ka case. In dono ka farq interview ka **classic trap** hai.

**Why this matters in production**:
- **Amazon** ka "Place Order" ek **idempotency token** generate karta hai jaise hi tum checkout page kholte ho — taake refresh ya double-click pe ek hi order bane.
- **Daraz/FoodPanda** order place karte waqt price aur stock ko order_items mein **snapshot (lock)** kar lete hain — taake kal price badle to tumhare confirmed order ka total na badle.
- **Stripe** ka pura API **idempotency-key** header pe bana hai: same key bhejo to woh purana result wapas deta hai, naya charge nahi karta.

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: `@Transactional` (atomicity), unique idempotency key (double-submit guard), server-side total recompute, aur **method overloading** for convenience entry points.

**Under the Hood — Yeh Kaise Kaam Karta Hai**:

Pehle `@Transactional` ka asli mechanism samjho — yeh "magic" nahi hai. Jab tum kisi `@Service` method pe `@Transactional` lagate ho, Spring **startup pe** us class ka ek **proxy** banata hai. Proxy = ek wrapper object jo asli object ke "aage khada" hota hai. Tum jab method call karte ho, dar-asal proxy ka method chalta hai jo yeh karta hai:

```
T+0: Proxy intercept karta hai — DataSource se ek DB connection uthata hai
T+1: connection.setAutoCommit(false) — ab har statement turant commit nahi hoga
T+2: Tumhara asli business method chalta hai (INSERT order, INSERT items, UPDATE stock...)
T+3a: Method bina exception ke khatam → proxy connection.commit() — sab ek saath pakka
T+3b: RuntimeException uchhla → proxy connection.rollback() — sab undo, jaise kuch hua hi nahi
T+4: connection wapas pool mein
```

Yeh "auto-commit off + commit/rollback at boundary" hi transaction ka dil hai. **Gotcha**: Spring by default sirf `RuntimeException` (unchecked) pe rollback karta hai, **checked exception pe NAHI**. Agar tum `IOException` (checked) throw karoge to transaction commit ho jayega! Isliye ya `RuntimeException` throw karo ya `@Transactional(rollbackFor = Exception.class)` likho.

Doosri gotcha — **self-invocation**: agar same class ke andar ek method doosre `@Transactional` method ko `this.otherMethod()` se call kare, to proxy bypass ho jata hai aur transaction lagta hi nahi (kyunki call proxy se nahi guzri, seedha object pe gayi).

**Bhai, Simple Mein Samjho**:
`@Transactional` woh **shaadi ka qazi** hai. Nikah ya to **pura** hota hai (dono "qabool hai" bolein → commit) ya bilkul nahi (ek ne mana kiya → sab cancel, rollback). Aadha nikah jaisi koi cheez nahi hoti. Transaction bhi aadha order nahi banने deta.

**Code Pattern**:
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;
    private final ProductRepository productRepo;
    private final CartRepository cartRepo;
    private final CartPricingService pricingService;   // Day 8 wala — total recompute

    // ---- OVERLOADING: same naam "placeOrder", alag parameters (compile-time) ----

    // 1) Sabse simple: bina coupon
    @Transactional
    public Order placeOrder(Long userId, String idempotencyKey) {
        return placeOrder(userId, idempotencyKey, null);   // null coupon ke saath delegate
    }

    // 2) Coupon ke saath — yeh "real" wala method, baaki ispe delegate karte hain
    @Transactional   // <-- saare 4 steps ek transaction mein
    public Order placeOrder(Long userId, String idempotencyKey, String couponCode) {

        // STEP 0: Idempotency guard — yeh request pehle to nahi aayi?
        Optional<Order> existing = orderRepo.findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            return existing.get();   // double-click — purana order wapas, naya NAHI
        }

        Cart cart = cartRepo.findByUserId(userId)
            .orElseThrow(() -> new EmptyCartException("Cart khaali hai"));
        if (cart.getItems().isEmpty())
            throw new EmptyCartException("Cart khaali hai");

        // STEP 1: Server-side total recompute (client ke total ko ignore!)
        DiscountStrategy discount = resolveCoupon(couponCode);     // null-safe
        CartTotals totals = pricingService.calculate(cart.getItems(), discount);

        // STEP 2: Order banao + items copy karo (price LOCK/snapshot)
        Order order = new Order(userId, idempotencyKey, OrderStatus.PLACED);
        for (CartLine line : cart.getItems()) {
            Product p = productRepo.findById(line.productId())
                .orElseThrow(() -> new ProductNotFoundException(line.productId()));

            // STEP 3: Stock kam karo — safe check (oversell se bachao)
            if (p.getStock() < line.quantity())
                throw new InsufficientStockException(p.getId());   // -> rollback sab
            p.setStock(p.getStock() - line.quantity());

            // price ko ABHI lock karo — kal price badle to is order pe asar na ho
            order.addItem(new OrderItem(p.getId(), line.quantity(), p.getPrice()));
        }
        order.setGrandTotal(totals.grandTotal());
        orderRepo.save(order);

        // STEP 4: Cart clear
        cart.clear();
        cartRepo.save(cart);

        return order;   // bina exception -> proxy COMMIT karega (sab ek saath)
    }

    private DiscountStrategy resolveCoupon(String code) {
        if (code == null) return new NoDiscount();   // Null Object pattern (Day 72 jhalak)
        // ... lookup + validity check
        return new FlatDiscount(new BigDecimal("100.00"));
    }
}
```

**Interview phrasing**:
"Iss scenario mein main `@Transactional` use karunga taake order-create, stock-decrement aur cart-clear **atomic** rahein — partial failure pe sab rollback ho. Idempotency key se double-submit guard karunga, aur total backend pe `CartPricingService` se recompute karunga, client ke total pe bharosa nahi."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: EF Core `SaveChanges` ka **implicit transaction**, ya explicit `BeginTransactionAsync`, plus **method overloading** — aur yahan ek **bohot bara Java vs C# farq**: C# mein methods **by default `virtual` nahi** hote (overriding ke liye `virtual`/`override` likhna parta hai), jabki Java mein har method default virtual hota hai.

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**:

EF Core (Entity Framework Core = .NET ka ORM, Java ke Hibernate jaisa) ek **change tracker** rakhta hai. Jab tum entity load karte ho ya add karte ho, EF har entity ki state yaad rakhta hai: `Added`, `Modified`, `Deleted`, `Unchanged`. Jab tum **ek hi** `SaveChangesAsync()` call karte ho, EF **automatically** ek transaction kholta hai, saare tracked changes ko sahi order mein SQL banata hai (INSERT/UPDATE/DELETE), aur sab ek transaction mein commit karta hai. Agar koi statement fail ho → poora rollback. Matlab **ek `SaveChanges` = ek atomic unit**, free mein.

Lekin agar tumhe multiple `SaveChanges` ya raw SQL ko ek atomic unit banana ho, to explicit `BeginTransactionAsync()` use karte ho.

**Bhai, .NET Mein Yeh Kaise Hota Hai**:
EF Core khud qazi ban jata hai jab tum sab kuch ek `SaveChanges` mein karte ho — woh implicit transaction de deta hai. Java mein hum `@Transactional` se boundary khud declare karte hain; C# mein chhote cases `SaveChanges` khud handle kar leta hai.

**Code Pattern**:
```csharp
[ApiController]
[Route("api/orders")]
public class OrderController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly CartPricingService _pricing;

    // OVERLOADING: do PlaceOrder, alag signatures (compile-time resolve)
    public Task<Order> PlaceOrder(long userId, string idempotencyKey)
        => PlaceOrder(userId, idempotencyKey, couponCode: null);

    public async Task<Order> PlaceOrder(long userId, string idempotencyKey, string? couponCode)
    {
        // ek transaction — saare 4 steps ya sab ya kuch nahi
        await using var tx = await _db.Database.BeginTransactionAsync();
        try
        {
            // idempotency guard
            var existing = await _db.Orders
                .FirstOrDefaultAsync(o => o.IdempotencyKey == idempotencyKey);
            if (existing != null) return existing;

            var cart = await _db.Carts.Include(c => c.Items)
                .FirstAsync(c => c.UserId == userId);

            var totals = _pricing.Calculate(cart.Items, ResolveCoupon(couponCode));
            var order = new Order(userId, idempotencyKey, OrderStatus.Placed);

            foreach (var line in cart.Items)
            {
                var p = await _db.Products.FindAsync(line.ProductId);
                if (p!.Stock < line.Quantity)
                    throw new InsufficientStockException(p.Id);
                p.Stock -= line.Quantity;                 // change tracker note karega
                order.AddItem(new OrderItem(p.Id, line.Quantity, p.Price));
            }
            order.GrandTotal = totals.GrandTotal;
            _db.Orders.Add(order);
            cart.Items.Clear();

            await _db.SaveChangesAsync();   // saare changes ek transaction
            await tx.CommitAsync();         // pakka
            return order;
        }
        catch
        {
            await tx.RollbackAsync();       // sab undo
            throw;
        }
    }
}
```

**Java vs .NET Comparison Table**:
| Feature | Java/Spring | .NET/C# |
|---------|-------------|---------|
| Transaction | `@Transactional` (proxy-based, declarative) | `SaveChanges` implicit / `BeginTransactionAsync` explicit |
| Default rollback | sirf unchecked exceptions | exception → tum khud rollback (try/catch) ya ambient `TransactionScope` |
| ORM | JPA / Hibernate | Entity Framework Core |
| Method **virtual** | **default virtual** (override allowed unless `final`) | **default non-virtual** (override ke liye `virtual` + `override` zaroori) |
| Async | `CompletableFuture` | `async/await` |
| Overloading resolve | compile-time, static arg types | compile-time, static arg types (same concept) |

> **Yeh farq yaad rakho — interview gold**: Java mein method ko override hone se rokne ke liye `final` lagate ho. C# mein **ulta** — by default koi override nahi kar sakta, allow karne ke liye `virtual` lagate ho. Yani "safe by default" philosophy alag hai.

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: `BEGIN TRANSACTION ... COMMIT/ROLLBACK`, ek **UNIQUE constraint** idempotency key pe (double-order ka asli DB-level bachaav), aur safe stock decrement.

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**:

Jab tum `BEGIN TRANSACTION` likhte ho, SQL engine har change ko pehle ek **transaction log** (SQL Server) / **redo+undo log** (MySQL InnoDB) mein likhta hai — **isse WAL kehte hain (Write-Ahead Log: pehle log mein likho, phir data file mein)**. Yeh isliye taake agar beech mein bijli chali jaye, restart pe DB log dekh ke ya to adhure transaction ko **undo** kar de, ya committed ko **redo**. `COMMIT` ka matlab: "log mein 'commit' record likh do aur disk pe flush kar do" — uske baad woh change permanent (durable) hai. `ROLLBACK` ka matlab: "undo log se sab wapas pehle jaisa kar do".

Ab **idempotency ka asli, bullet-proof tareeqa**: `orders` table mein `idempotency_key` pe ek **UNIQUE constraint**. Application-level `if (exists)` check mein race condition reh sakti hai (do request bilkul same millisecond pe), lekin **UNIQUE index** DB level pe guarantee hai — doosra INSERT **fail** ho jayega (duplicate key error), chahe kitne bhi concurrent ho. Yeh "last line of defense" hai.

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**:
UNIQUE constraint woh **darbaan** hai jo bolta hai "ek idempotency-key pe ek hi order, bas". Application check toot bhi jaye, darbaan nahi torta. Aur transaction woh **register** hai jisme ya pura nikah likha jata hai ya page hi phaad diya jata hai — koi aadhi entry nahi.

**SQL Example**:
```sql
-- Schema: idempotency_key pe UNIQUE = DB-level double-order guard
ALTER TABLE orders
    ADD CONSTRAINT uq_orders_idemp UNIQUE (idempotency_key);

-- Place order = ek atomic transaction
BEGIN TRANSACTION;

    -- 1) Order banao (agar idempotency_key duplicate hua to yeh INSERT fail -> rollback)
    INSERT INTO orders (user_id, idempotency_key, status, grand_total)
    VALUES (@userId, @idempKey, 'PLACED', @grandTotal);

    SET @orderId = SCOPE_IDENTITY();   -- abhi bana order ka id

    -- 2) Stock SAFE decrement: WHERE stock >= qty (oversell guard)
    UPDATE products
       SET stock = stock - @qty
     WHERE id = @productId
       AND stock >= @qty;             -- agar stock kam hua to 0 rows update

    IF @@ROWCOUNT = 0                   -- stock nahi mila -> abort
    BEGIN
        ROLLBACK TRANSACTION;          -- order + jo bhi hua, sab undo
        THROW 50001, 'Insufficient stock', 1;
    END

    -- 3) order_items mein price LOCK karke daalo
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    SELECT @orderId, @productId, @qty, price FROM products WHERE id = @productId;

    -- 4) Cart clear
    DELETE FROM cart_items WHERE cart_id = @cartId;

COMMIT TRANSACTION;   -- sab ek saath pakka
```

**The Gotcha**:
Agar tum `UPDATE products SET stock = stock - @qty` **bina** `AND stock >= @qty` likho, to stock **negative** ho sakta hai (-5 phone!) = oversell. Aur agar UNIQUE constraint na ho, to do parallel requests dono `INSERT` kar dengi = duplicate order. Dono guards zaroori hain.

**Isolation Level Choice**:
Simple version ke liye default **READ COMMITTED** (har read sirf committed data dekhe) kaafi hai, kyunki humne stock guard `AND stock >= @qty` se atomic conditional update bana diya — DB row-lock khud le lega us UPDATE ke dauraan. High-contention (flash sale) pe `UPDLOCK`/pessimistic ya optimistic version column lagta hai — woh **Day 62 (optimistic vs pessimistic locking)** ka topic hai.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Double-click se bachne ke liye button ko **disable + in-flight guard**, idempotency key client pe generate karna, aur `HttpClient` ke saath safe submit + error rollback.

**Under the Hood — Angular Yeh Kaise Karta Hai**:

Angular ka **change detection** har async event (click, HTTP response) ke baad component tree ko scan karta hai aur view ko update karta hai — yeh kaam **Zone.js** karta hai. Zone.js browser ke async APIs (`addEventListener`, `setTimeout`, `XHR`) ko "monkey-patch" karta hai (matlab unko apne wrapper se replace karta hai) taake jab koi async kaam khatam ho to Angular ko pata chal jaye "ab UI refresh karna hai". Isi liye jab tumhara HTTP order-call return karta hai, button ka `disabled` state automatically UI mein reflect ho jata hai — tumhe manually DOM touch nahi karna parta.

Idempotency key client pe `crypto.randomUUID()` se banao **jab page load ho** (har checkout attempt ke liye ek), aur usse retry/double-click pe bhi **same** key bheja jaye — taake backend pehchaan le "yeh wahi request hai".

**Bhai, Frontend Pe Yeh Kaise Handle Karein**:
Soch FoodPanda pe "Place Order" daba ke 2 second kuch nahi hua — banda ghusse mein 5 baar dabata hai. Agar button disable na ho, 5 request jayengi. Solution: **pehle click pe button disable + spinner**, request complete (success ya fail) hone tak. Plus same idempotency key — agar koi request nikal bhi gayi to backend duplicate na bane.

**Code Pattern**:
```typescript
@Component({ /* ... */ })
export class CheckoutComponent {
  placing = false;                              // in-flight guard
  // ek checkout attempt = ek idempotency key (page load pe banaya)
  private readonly idempotencyKey = crypto.randomUUID();

  constructor(private orderApi: OrderService, private router: Router) {}

  placeOrder(): void {
    if (this.placing) return;                   // pehle se chal raha -> ignore extra clicks
    this.placing = true;                        // button disable (template: [disabled]="placing")

    this.orderApi.placeOrder(this.idempotencyKey).subscribe({
      next: (order) => this.router.navigate(['/orders', order.id]),  // success -> order page
      error: (err) => {
        this.placing = false;                   // wapas enable, user retry kar sake
        this.showError(err);                    // friendly message
      }
      // NOTE: success pe placing true hi rehne dete hain (navigate ho raha hai)
    });
  }
}
```
```typescript
@Injectable({ providedIn: 'root' })
export class OrderService {
  constructor(private http: HttpClient) {}

  placeOrder(idempotencyKey: string): Observable<Order> {
    return this.http.post<Order>('/api/orders',
      {},                                       // body — server cart khud uthayega
      { headers: { 'Idempotency-Key': idempotencyKey } }   // <-- key header mein
    );
  }
}
```
```html
<!-- button khud disable, double-click impossible -->
<button (click)="placeOrder()" [disabled]="placing">
  {{ placing ? 'Placing...' : 'Place Order' }}
</button>
```

**UX Concern**:
Bina `placing` guard ke: user frustration mein multiple click → multiple orders → support tickets → refunds. Stripe/Amazon isiliye button ko **instantly** disable karte hain aur spinner dikhate hain. "Optimistic UI" (turant success dikha do) yahan **galat** choice hai — order/payment irreversible hai, isliye **pessimistic** raho: jab tak server confirm na kare, success mat dikhao.

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: **Transactional order creation** (single service, ACID) + **idempotency key** + (next-level) **event publish** for downstream (email, inventory sync) — synchronous core, asynchronous side-effects.

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**:

Beginner level pe order ek **single service + single DB transaction** mein banta hai — yeh **ACID** deta hai (Atomicity, Consistency, Isolation, Durability). Lekin order ke baad bohot kuch hota hai: confirmation email, SMS, warehouse ko notify, analytics. In sab ko **synchronously** (order request ke andar) karna ghalat hai — agar email service down hai to kya order fail kar doge? Nahi!

Isliye core order **commit** hone ke baad ek **event publish** hota hai (`OrderPlaced`), aur baaki services (email, inventory, analytics) us event ko **asynchronously** consume karte hain. Order placement fast aur reliable rehta hai; side-effects independently retry hote hain. (Async events ka real depth — Kafka, dead-letter — Day 70+ pe.)

**Bhai, Architecture Level Pe Yeh Kaise Design Karein**:
Order banana = **must succeed now** (synchronous, transactional). Email bhejna = **eventually**, alag se. Inko mat mix karo. Shaadi ka nikah (core) abhi pakka karo; mehndi/dawat (side-effects) baad mein bhi ho sakti hai.

**Architecture Diagram**:
```
┌─────────────┐    ┌────────────────────┐    ┌──────────────┐
│   Angular   │───▶│  OrderService      │───▶│   SQL DB     │
│ Checkout    │    │  (Spring / .NET)   │    │  orders      │
│             │    │                    │    │  order_items │
└─────────────┘    └────────────────────┘    │  products    │
   │ disable btn        │ @Transactional      │  (1 TXN)     │
   │ Idempotency-Key    │ idempotency guard   └──────────────┘
   │ (UUID)             │ server recompute          │ ACID + UNIQUE key
   │                    │                           │
   │                    ▼ (AFTER commit)            │
   │            ┌────────────────┐                  │
   │            │  OrderPlaced   │  async           │
   │            │  event/queue   │──▶ Email/SMS/Inventory
   │            └────────────────┘                  │
   └────────────────────┴──────────────────────────┘
                  Consistent Order State
```

**Trade-offs**:
| Approach | Pros | Cons |
|----------|------|------|
| Single TXN, sab synchronous (incl. email) | Simple, strongly consistent | Email/SMS down → order fail; slow response |
| TXN for core + async events for side-effects | Fast, resilient, scalable | Eventual consistency; need queue + retry infra |
| Distributed TXN across services (2PC) | Strong consistency cross-service | Slow, complex, locks held long — avoid (Saga better, Day 51) |

**Real Companies Using This**:
- **Amazon**: order commit synchronous + downstream (warehouse, email, recommendations) event-driven.
- **Stripe**: idempotency-key driven — same key = same result, no double charge.
- **FoodPanda**: order placed atomically, phir rider-assignment/restaurant-notify async events.

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: **Method Overloading vs Method Overriding**

### Principle/Pattern Definition

**Concept**:
- **Overloading** = **same method naam, alag parameters** (number/type/order). Same class mein. Compiler decide karta hai kaunsa chalega — **compile-time** pe, argument ke **static types** dekh ke. Isse **static (early) binding** kehte hain.
- **Overriding** = **child class** parent ke method ko **same signature** ke saath dobara likhti hai (behavior badalne ke liye). JVM/CLR decide karta hai kaunsa chalega — **run-time** pe, **actual object** dekh ke. Isse **dynamic (late) binding** kehte hain. Yeh **polymorphism** ka engine hai.

**Bhai, Simple Mein Samjho**:
- **Overloading** = ek hi naam ke do alag kaam jo **arguments se** pehchaane jaate hain. Jaise `print(int)` aur `print(String)` — naam same, kaam thoda alag, compiler likhte waqt hi decide kar leta hai.
- **Overriding** = beta baap ka kaam **apne tareeqe se** karta hai. Naam aur signature bilkul same, lekin andar ka behavior badla. Faisla **run-time** pe hota hai — jo object asal mein hai, uska version chalega.

**Real-Life Analogy (Pakistani Context)**:
- **Overloading** = **"Chai banao" with options.** `chai()` = normal doodh-patti. `chai(chini)` = itni chini ke saath. `chai(chini, ilaichi)` = chini + ilaichi. Naam same "chai", lekin **jo cheezein do** (parameters) us hisaab se ban-ti hai — aur tum **order dete waqt hi** (compile-time) decide kar lete ho kaunsi chai.
- **Overriding** = **Careem ride types.** Base class `Ride` mein `calculateFare()` hai. `Bike`, `Go`, `Plus` sab `Ride` extend karte hain aur `calculateFare()` ko **apne hisaab se override** karte hain (Bike sasti, Plus mehngi). Jab tum `Ride r = bookSomething()` karte ho, jo asli object aaya (Bike ya Plus) **uska** fare chalega — **run-time** pe decide.

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Order placement mein dono saaf nazar aate hain:

1. **Overloading** — `placeOrder(userId, key)`, `placeOrder(userId, key, couponCode)`, aur shayad `placeOrder(userId, key, couponCode, addressId)` for guest/saved-address checkout. Naam ek, parameters alag. Caller jo arguments deta hai, compiler **likhte waqt** hi sahi version bind kar deta hai. Simple version dusre, full version pe **delegate** karta hai (`return placeOrder(userId, key, null)`).

2. **Overriding** — Order place hone ke baad notification: base class `OrderNotification` ka `send(Order)` method, jise `EmailNotification`, `SmsNotification`, `WhatsAppNotification` **override** karte hain. Tum `OrderNotification n = user.preferredChannel()` likhte ho aur `n.send(order)` call karte ho — **actual object** ka version run-time pe chalega. Yeh hi polymorphism hai.

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

**❌ BAD (dono ko gaddmadd / galat istemaal)**:
```java
// Overloading ke naam pe confusion — sirf return type ya bina-logic ke copy
public class OrderService {
    // Yeh OVERLOADING NAHI hai — Java sirf return type pe overload allow NAHI karta!
    // Compile error: "method already defined"
    public Order process(Long id) { /* ... */ }
    public String process(Long id) { /* ... */ }   // ❌ same params, sirf return type alag

    // Aur har coupon type ke liye alag method naam — ugly, non-polymorphic
    public Order placeOrderWithFlat(Long id) { /* duplicate logic */ }
    public Order placeOrderWithPercent(Long id) { /* duplicate logic */ }  // copy-paste!
}
```
```java
// Overriding ka galat istemaal — child ne contract tod diya
class EmailNotification extends OrderNotification {
    @Override
    public void send(Order o) {
        throw new UnsupportedOperationException();   // ❌ LSP violation (Day 13 jhalak)
    }
}
```

**✅ GOOD (sahi overloading + sahi overriding)**:
```java
// OVERLOADING: same naam, alag params, ek dusre pe delegate (no duplication)
public Order placeOrder(Long userId, String key) {
    return placeOrder(userId, key, null);                 // delegate
}
public Order placeOrder(Long userId, String key, String couponCode) {
    return placeOrder(userId, key, couponCode, null);     // delegate
}
public Order placeOrder(Long userId, String key, String couponCode, Long addressId) {
    // ASLI logic sirf yahan — baaki sab convenience overloads
    /* ... transaction, recompute, stock ... */
}
```
```java
// OVERRIDING: base contract, har channel apna behavior, run-time dispatch
abstract class OrderNotification {
    abstract void send(Order order);             // contract
}
class EmailNotification extends OrderNotification {
    @Override void send(Order order) { /* SMTP se email */ }
}
class SmsNotification extends OrderNotification {
    @Override void send(Order order) { /* Twilio se SMS */ }
}

// Caller — polymorphism: actual object ka send() chalega (run-time)
OrderNotification notifier = user.preferredChannel();   // Email ya Sms
notifier.send(order);                                    // dynamic dispatch
```

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Overloading | same naam, alag params (same class) | bilkul same concept |
| Overload on return type only | ❌ allowed nahi | ❌ allowed nahi |
| Override keyword | `@Override` (optional, recommended) | `override` (**mandatory**) |
| Methods virtual by default? | **Haan** (override allowed unless `final`) | **Nahi** (parent pe `virtual` zaroori) |
| Prevent override | `final void m()` | non-virtual (kuch na likho) ya `sealed override` |
| Hide instead of override | (rare) | `new` keyword (method hiding) |

```csharp
// C# — note: virtual + override DONO zaroori (Java se bara farq)
public abstract class OrderNotification {
    public abstract void Send(Order order);          // abstract = virtual + must override
}
public class EmailNotification : OrderNotification {
    public override void Send(Order order) { /* SMTP */ }   // 'override' likhna mandatory
}

// Overloading — same as Java
public Order PlaceOrder(long userId, string key)
    => PlaceOrder(userId, key, couponCode: null);
public Order PlaceOrder(long userId, string key, string? couponCode) { /* asli logic */ }
```
> **C# ka `new` (method hiding) trap**: agar tum child mein `override` ki jagah `new void Send()` likho, to woh parent ko override nahi karta — **hide** karta hai. Phir `OrderNotification n = new EmailNotification(); n.Send()` **parent** ka version chalayega (reference type dekh ke), na ke child ka! Yeh C# ka famous gotcha hai.

---

### 🎯 Cross-Questioning Drill (Interview Defense)

**Cross Q1**: "Overloading aur overriding mein asli farq kya hai — ek line mein?"
**Confident Answer**:
"Bhai, ek line: **Overloading compile-time pe argument ke static types se resolve hota hai (static binding); overriding run-time pe actual object se resolve hota hai (dynamic binding).** Overloading 'same naam alag params, same class' hai — convenience ke liye. Overriding 'same signature, child class' hai — polymorphism ke liye. Isiliye overloading ko aksar 'compile-time polymorphism' aur overriding ko 'run-time polymorphism' kehte hain, halanki purists overloading ko sachcha polymorphism nahi maante."

---

**Cross Q2**: "Overload resolution actual object se hota hai ya reference type se? Ek trick example do."
**Confident Answer**:
"Reference (static) type se — yeh log phasते hain. Dekho:
```java
void log(Object o) { System.out.println("Object"); }
void log(String s) { System.out.println("String"); }

Object x = "hello";
log(x);   // prints "Object" — NOT "String"!
```
Halanki `x` asal mein String hai, lekin uska **declared type** `Object` hai, aur overload **compile-time** pe declared type se resolve hota hai → `log(Object)` chalta hai. Agar yeh overriding hota to actual object (String) wala chalta. **Yahi overloading vs overriding ka core farq hai.**"

---

**Cross Q3**: "Tum overloading kab use karoge aur kab nahi (alternative kya)?"
**Confident Answer**:
"Overloading tab jab same conceptual operation ho lekin convenience ke liye alag inputs — jaise `placeOrder` with/without coupon. Lekin agar behaviour kaafi alag ho jaye, to overloading ki jagah **alag naam wale methods** ya **Builder/parameter-object** behtar hai. Java mein `Optional` params nahi hote isiliye overloading common hai; C# mein **optional parameters** (`couponCode = null`) aur **named arguments** hote hain, to wahan kabhi-kabhi overloading ki zaroorat hi nahi parti. Bohot saare overloads = code smell — tab parameter object ya Builder use karo."

---

**Cross Q4**: "Yeh principle Spring/EF mein automatically aata hai ya manual?"
**Confident Answer**:
"Overriding to frameworks ki **buniyaad** hai. Spring ka `@Transactional` proxy dar-asal tumhari class **extend** karke methods **override** karta hai (CGLIB proxy) — isiliye `final` methods proxy nahi ho paate aur `private`/`final` pe `@Transactional` kaam nahi karta! Hibernate lazy-loading bhi proxy subclass se overriding pe chalti hai. Spring `WebMvcConfigurer` jaise interfaces ke default methods ko hum override karte hain config ke liye. Yani overriding manual bhi hai aur framework ke andar bhi har jagah. Overloading zyada-tar humari convenience APIs mein hota hai."

---

**Cross Q5**: "Production/scale pe in dono ka koi performance ya design impact hai?"
**Confident Answer**:
"Performance pe overriding ka cost na-ke-baraabar hai — JVM ka **vtable** (virtual method table: har class ka function-pointer array) ek extra indirection deta hai, lekin JIT compiler aksar **monomorphic** call sites ko inline kar deta hai, to real cost almost zero. Overloading ka koi run-time cost hai hi nahi (compile-time resolved). **Design impact zyada matters**: overriding loose coupling aur extensibility deta hai (naya notification channel = naya subclass, purana code na chhuo — Open/Closed). Overloading APIs ko ergonomic banata hai. Scale pe asli faisla 'overriding vs strategy/composition' ka hota hai — deep inheritance trees maintainability maar dete hain, isiliye senior log aksar overriding ki jagah **composition + Strategy pattern** prefer karte hain."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

**Pairs Well With**:
- **Strategy Pattern** — kyunki overriding hi to Strategy ko chalata hai (har strategy `apply()` override karti hai). Day 8 ka `DiscountStrategy` isi pe khada tha.
- **Template Method (Day 66)** — kyunki base class skeleton deta hai, child **specific steps override** karta hai.
- **Factory Method (Day 32)** — kyunki factory aksar object banane ka method **override** karwati hai.

**Conflicts With / Tension**:
- **Deep Inheritance** — bohot zyada overriding = bhari inheritance tree = "fragile base class" problem (base badlo, sab child toot-te hain). Tab **composition** behtar.
- **Optional parameters (C#)** — bohot saare overloads tab redundant ho jaate hain jab language optional/named params deti ho.

---

### 🎓 Real Production Code Where This Matters

`java.util.List.add()` overloading ka perfect example hai:
- `add(E element)` — end pe daalo
- `add(int index, E element)` — kisi position pe daalo

Same naam, alag params, compiler decide karta hai. Aur overriding: `ArrayList`, `LinkedList` dono `List.add()` ko **apne tareeqe se override** karte hain (ArrayList array shift karta hai, LinkedList pointer jod-ta hai). Jab tum `List<Order> orders = new LinkedList<>()` likh ke `orders.add(o)` karte ho, **LinkedList wala** version chalta hai — run-time dispatch. Spring Data ke `JpaRepository` ke `save()` ko bhi implementation override karti hai.

---

### 💡 Memory Hook for This Principle/Pattern

- **Overloading** = **"NAAM EK, SAMAAN ALAG, FAISLA PEHLE"** (same naam, alag arguments, compiler likhte-waqt decide → compile-time).
- **Overriding** = **"BAAP KA KAAM, BETE KA TAREEQA, FAISLA BAAD MEIN"** (same signature, child ka behavior, run-time pe actual object decide).
- One-shot: **"OverLOADING = compile-time (LOAD karte waqt likhte ho), OverRIDING = run-time (RIDE pe nikalte waqt pata chalta hai kaun chala raha hai)."**

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

**❌ Mistake 1**: "Return type alag karke overload ban jata hai."
**Why it's wrong**: Java/C# dono **return type pe overload allow nahi** karte — compiler signature mein return type count hi nahi karta. `Order process(Long)` aur `String process(Long)` = compile error.
**Correct approach**: Overload ke liye **parameters** (number/type/order) badlo, return type nahi.

**❌ Mistake 2**: "Override pe `@Override`/`override` likhna optional hai, chhoro."
**Why it's wrong**: Bina annotation ke agar signature thoda galat ho (e.g., `send(Orders)` likh diya `send(Order)` ki jagah), to woh **chup-chaap overload ban jayega, override nahi** — aur tumhara polymorphism toot jayega, koi error bhi nahi aayega. C# mein to `override` mandatory hai, Java mein `@Override` lagao taake compiler verify kare.
**Correct approach**: Hamesha `@Override` (Java) / `override` (C#) likho — compiler galti pakad lega.

**❌ Mistake 3**: "Overloading actual object dekh ke resolve hoti hai (overriding ki tarah)."
**Why it's wrong**: Overloading **declared/static type** dekh ke compile-time pe resolve hoti hai. `Object x = "hi"; print(x)` → `print(Object)` chalega, na ke `print(String)`. Yeh sabse common interview trap hai.
**Correct approach**: Yaad rakho — overloading = static type, compile-time; overriding = actual object, run-time.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ "Place Order" click → button disable + spinner
       └─▶ Idempotency-Key (UUID) header ke saath POST /api/orders

2. API REQUEST (Spring Boot / .NET)
   └─▶ placeOrder() [OVERLOADED] → @Transactional boundary
       └─▶ Idempotency guard: key pehle aayi? → purana order return

3. BUSINESS LOGIC (Java / C#)
   └─▶ Cart recompute (server-side, client total ignore)
       └─▶ Stock check + decrement, price snapshot in order_items

4. DATABASE (SQL)
   └─▶ BEGIN TXN → INSERT order (UNIQUE idemp key) → UPDATE stock (>= guard)
       → INSERT items → DELETE cart → COMMIT (atomic)

5. EVENT PUBLISHING (System Design)
   └─▶ AFTER commit → OrderPlaced event → Email/SMS notify [OVERRIDDEN send()]

6. RESPONSE (All layers)
   └─▶ DB commit → API 200 + order → Angular → /orders/:id page
```

**What Breaks If You Skip ANY Layer**:
- **Angular guard hata do** → double-click → multiple submits (idempotency key bachata hai, par UX kharab).
- **@Transactional hata do** → crash pe partial order: stock kata par order gayab, ya cart clear par order nahi.
- **Server-side recompute hata do** → user DevTools se total `1` bhej ke loot le jaye.
- **SQL UNIQUE key + stock guard hata do** → duplicate orders + negative stock (oversell).
- **Sync email rakho (event na karo)** → email service down = order fail; ya slow response.

---

## 🧭 MENTAL MAP — How to Memorize This

```
                [PLACE ORDER]
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    [Frontend]   [Backend]      [Database]
        │             │             │
    Angular     Spring/.NET        SQL
        │             │             │
    disable btn  @Transactional  BEGIN TXN
    Idemp-Key    idemp guard     UNIQUE key
    spinner      recompute total stock>=qty guard
    pessimistic  overload entry  COMMIT/ROLLBACK
        │             │             │
        └─────────────┼─────────────┘
                      │
              [SYSTEM DESIGN]
        TXN core (ACID) + async OrderPlaced event
              (notify via OVERRIDDEN send())
```

**Mental Story to Remember (Roman Urdu)**:
"Socho tum **shaadi ka nikah** kara rahe ho (= place order):
- Angular = **dulha jo ek hi baar 'qabool hai' bole** (button disable, double-qabool nahi).
- Spring/.NET = **qazi** jo nikah ya pura karta hai ya bilkul nahi (`@Transactional`).
- Java/C# = **gawah** jo confirm karte hain sab sahi hai (recompute, stock check).
- SQL = **nikah-nama register** — ek key pe ek hi entry (UNIQUE), aadhi entry nahi (TXN).
- System Design = **dawat ke cards** baad mein bhejo (async OrderPlaced event), nikah ke saath mat atkao.
Agar koi missing — ya do nikah ho jayenge, ya register adhura, ya dawat ke chakkar mein nikah ruk jaye."

**Acronym/Mnemonic**: **"TIRES"** = **T**ransaction (atomic) → **I**dempotency (no double) → **R**ecompute (server total) → **E**vents (async side-effects) → **S**tock-guard (no oversell). *Order place karne ke liye gaadi ke TIRES sahi hone chahiye.*

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Design the 'Place Order' flow for an e-commerce cart. User clicks once, order banna chahiye — kaise ensure karoge ke double-click pe do order na banein, partial failure pe data inconsistent na ho, aur total se cheating na ho?"

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Iss flow mein 5 layers coordinate karenge, aur teen guarantees deni hain: **atomicity, idempotency, aur server-side truth**."

**Body** (60 sec):
1. **Frontend** (Angular): Click pe button instantly disable + spinner (pessimistic UI), aur ek **idempotency key (UUID)** header mein bhejo — page load pe banaya, retry pe same.
2. **Backend API** (Spring/.NET): `placeOrder()` ko `@Transactional` mein wrap karo. Pehle idempotency-key DB mein check karo — agar exist karti hai to **purana** order return karo, naya nahi.
3. **Business Logic** (Java/C#): Cart ko server pe **dobara** uthao, total **recompute** karo (client total ignore). Stock check + decrement, price ko order_items mein **snapshot** karo.
4. **Database** (SQL): Sab kuch ek transaction mein — `idempotency_key` pe **UNIQUE constraint** (DB-level double guard) aur stock decrement pe `WHERE stock >= qty` (oversell guard). Fail → ROLLBACK.
5. **Architecture**: Order **commit** ke baad `OrderPlaced` **event** publish karo — email/SMS/inventory **async** handle karein, core flow ko block na karein.

**Closing** (10 sec):
"In layers ko transaction + idempotency key + server-side recompute ke saath combine karke, hum guarantee karte hain: exactly ek order, consistent state, aur tamper-proof total."

### Under-the-Hood Concepts You MUST Know

1. **`@Transactional` proxy**: Spring startup pe proxy banata hai jo autoCommit off karke method ke boundary pe commit/rollback karta hai; default sirf unchecked exceptions pe rollback; self-invocation proxy bypass kar deti hai.
2. **WAL (Write-Ahead Log)**: DB pehle change log mein likhta hai phir data file mein — crash recovery (redo committed, undo uncommitted) aur durability ka basis.
3. **Idempotency**: Same request N baar = ek hi side-effect. UNIQUE constraint = DB-level last line of defense, app check race-prone hai.
4. **Static vs Dynamic binding**: Overloading compile-time pe declared type se; overriding run-time pe actual object se (vtable dispatch).
5. **Conditional atomic update**: `UPDATE ... SET stock = stock - q WHERE stock >= q` ek hi statement mein check+update — row lock automatically lega, `@@ROWCOUNT`/affected-rows se success pata.

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "Order ke baad email synchronous bhejo ya asynchronous? Trade-off?"
**Your Answer**:
"Asynchronous, bhai. Synchronous bhejo to email/SMS provider down hone pe **order hi fail** ho jayega — yeh galat hai, order to ban chuka, payment ho chuki. Plus SMTP slow hai, response 2-3 sec lag jayega. Async mein: order commit karo, phir `OrderPlaced` event queue mein daalo, email-service alag se consume kare aur fail pe **retry** kare. Trade-off: ab **eventual consistency** hai (email thodi der baad aayega) aur queue infra (Kafka/RabbitMQ) chahiye. Lekin core order fast aur resilient. Edge case: event publish khud transaction ke andar ho to 'dual-write problem' aata hai — uska solution **Transactional Outbox pattern** hai (event ko usi DB transaction mein ek outbox table mein likho, alag publisher uthaye)."

---

**Counter Q2 (Scale focused)**: "Flash sale — 50,000 log ek saath aakhri 100 units khareedna chahte hain. Tumhara simple stock decrement chalega?"
**Your Answer**:
"10x pe mera `UPDATE ... WHERE stock >= qty` chalega kyunki woh atomic conditional hai — DB row lock le lega, sirf 100 successful honge, baaki ko 0 rows → 'sold out'. Lekin 50,000 concurrent pe woh **ek hi row** pe sab lock ke liye lline lagayenge — **hot row contention**, throughput gir jayega aur latency spike. 100x-1000x pe solutions: (1) stock ko **Redis** mein rakho aur atomic `DECRBY` se reserve karo, DB ko async sync karo; (2) **inventory sharding** — 100 units ko 10 buckets mein baanto taake lock spread ho; (3) **queue-based** — sab requests ko queue mein daalo, ek consumer serially process kare. Yeh sab **Day 62 (locking)** aur **Day 69 (distributed locks)** ke topics hain. Beginner level pe atomic conditional update kaafi sahi default hai."

---

**Counter Q3 (Failure mode focused)**: "Order commit ho gaya lekin response client tak nahi pohncha (network drop). User refresh karta hai — kya hoga?"
**Your Answer**:
"Yahi pe idempotency key bachati hai. User ne jo UUID key bheji thi, woh **same** rehti hai (page load pe banayi, client storage mein). Refresh/retry pe wahi key dobara jaati hai. Backend dekhta hai 'yeh key pe order to already exist karta hai' → woh **wahi purana order** return kar deta hai, naya nahi banata. User ko apna confirmed order dikh jata hai, double-charge nahi hota. Bina idempotency key ke yeh scenario double order banata. Production mein key ko response milne tak persist rakhna important hai (sessionStorage/server-side checkout token)."

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Order place karte waqt price kis waqt ki use karoge — cart mein add karte waqt wali, ya place karte waqt wali?"
**Junior Answer**: "Jo cart mein hai woh use kar lo" — ya — "Jo abhi DB mein hai woh." (bina trade-off soche)
**Senior Answer**: "Yeh ek **business + consistency** decision hai. Best practice: place order ke waqt **current price** uthao (cart stale ho sakti hai — banda 3 din se cart mein chhor ke baitha hai), lekin us price ko `order_items` mein **snapshot/lock** kar lo taake order ban-ne ke baad price change ka asar na ho. Agar price add-to-cart se zyada ho gayi to **user ko confirm karwana** chahiye ('price badal gayi, phir bhi order karoge?') — silently zyada charge karna trust torta hai aur kayi jagah legally galat hai. Amazon yahi karta hai: cart mein price indicative, checkout pe re-validate, order pe lock. Yani teen alag jagah price ki teen alag responsibility — yeh distinction senior soch dikhati hai."

### Red Flag Signals (Don't Say These!)

- ❌ "Bas `INSERT INTO orders` kar do, ho gaya." — Why: atomicity, idempotency, stock, recompute — kuch nahi socha; production mein turant data corruption.
- ❌ "Frontend total bhej dega, backend save kar lega." — Why: massive security hole; user koi bhi total bhej sakta hai.
- ❌ "Double-click? Bas button disable kar do, kaafi hai." — Why: network retry/refresh button disable se nahi rukta; server-side idempotency key + UNIQUE constraint zaroori.
- ❌ "Overloading aur overriding same cheez hai." — Why: fundamental OOP galti; ek compile-time, doosra run-time — interviewer turant junior tag laga dega.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ `@Transactional` internally kaise atomicity deta hai (proxy + autocommit off + commit/rollback boundary) aur uske 2 gotchas (checked exception, self-invocation).
2. ✅ Idempotency kya hai aur double-submit ko 3 layers pe kaise rokte hain (button disable + idempotency key + DB UNIQUE constraint).
3. ✅ Order place karte waqt total/price server pe kyun recompute aur snapshot karte hain.
4. ✅ Stock oversell se bachne wala atomic conditional update (`WHERE stock >= qty`) kaise kaam karta hai.
5. ✅ Overloading vs Overriding ka core farq (compile-time/static binding vs run-time/dynamic binding) + Java vs C# ka `virtual` default difference.

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 10 — View Order History** (OOP: Static vs Instance members).

Aaj humne order **banaya**; kal user apne saare purane orders **dekhega** — pagination, sorting (latest first), aur sirf apne hi orders (ownership check). Aaj ke `orders`/`order_items` schema aur snapshot ki hui price kal history dikhane ke kaam aayegi. Aur OOP mein **static vs instance** — jaise ek `OrderStatus` constants (static) vs har order ka apna data (instance).

---

## 📚 Progress Tracker

```
🟢 Beginner     █████████░░░░░░░░░░░  Day 9/20   ← CURRENT
🟡 Intermediate ░░░░░░░░░░░░░░░░░░░░  Day 0/30
🟠 Advanced     ░░░░░░░░░░░░░░░░░░░░  Day 0/40
🔴 Expert       ░░░░░░░░░░░░░░░░░░░░  Day 0/50
⚫ Master       ░░░░░░░░░░░░░░░░░░░░  Day 0/60
🟣 Architect    ░░░░░░░░░░░░░░░░░░░░  Day 0/70
💎 Principal    ░░░░░░░░░░░░░░░░░░░░  Day 0/95
```

**Overall**: Day 9 of 365 (2.5% complete) · 🔥 9-day streak
