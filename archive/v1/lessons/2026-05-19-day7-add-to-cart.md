# 🎯 🟢 Day 7 of Beginner (Level 1 of 7): Add to Cart Functionality

**Overall Day**: Day 7 of 365
**Current Level**: 🟢 Beginner
**Level Progress**: Day 7 of 20
**Today's Theme**: FoodPanda/Daraz-style "Add to Cart" jahan user button daba ke item cart mein daale — duplicate na bane, quantity sahi merge ho, stock check ho, aur 100 log ek hi waqt last "iPhone" cart mein daalein to system pagal na ho.

---

## 📖 The Brainer Scenario (Real Problem)

Bhai, kal humne Daraz pe products **dhoonde** (search + pagination). Aaj user ko woh product **cart mein daalna** hai. Lagta simple hai — ek button, ek API call, item add. Lekin production mein yeh "Add to Cart" ek minefield hai.

Socho yeh flow:
1. User product page pe "Add to Cart" dabata hai → product ID + quantity backend ko jaata hai.
2. Agar wahi product pehle se cart mein hai, to **naya row nahi banna chahiye** — quantity merge honi chahiye (1 + 1 = 2, do alag rows nahi).
3. Stock check: agar sirf 3 pieces bache hain aur user 5 maange — error ya clamp.
4. **Guest user** (login nahi kiya) ka cart bhi chahiye — woh login kare to guest cart logged-in cart se **merge** ho.
5. Flash sale: 11:00 baje 500 log ek hi "iPhone 15" (sirf 10 stock) cart mein daal rahe hain — system crash na ho, oversell na ho.

**The Real Challenge (The "Gotcha")**:
- **Race condition on duplicate add**: User ne fast double-click kiya (ya 2 tabs khuli hain). Dono requests parallel aayi, dono ne dekha "cart mein item nahi hai", dono ne INSERT kiya → **2 duplicate rows**. Classic check-then-act bug.
- **Stock decrement kab karein?** Add-to-cart pe stock kam karo to log carts mein daal ke chhod dete hain → inventory artificially "sold out" dikhega (cart abandonment 70% hota hai!). Checkout pe karo to oversell ka risk.
- **Cohesion ka sawaal**: Naya developer `CartService.addToCart()` mein cart logic + pricing calculation + inventory update + email confirmation + analytics — sab thoonk deta hai. Yeh **God Service** ban jaata hai. Aaj ka OOP lesson exactly yahi hai: **Cohesion**.

**Why this matters in production**:
- **Amazon** cart ko stock se decouple karta hai — add-to-cart par sirf "availability snapshot" dikhata hai, actual reserve checkout pe karta hai with a short hold.
- **FoodPanda** guest cart cookie/Redis mein rakhta hai, login pe DB cart se merge karta hai.
- **Shopify** cart ko ek alag bounded service rakhta hai — cart service sirf cart jaanta hai, pricing/tax/inventory alag services hain (high cohesion at service level).

---

## 🔧 Stack 1: Java / Spring Boot (Backend Logic)

**Concept needed**: Atomic "upsert" of cart item via `@Transactional` service + unique constraint + a **highly cohesive** `CartService` (sirf cart ka kaam).

**Under the Hood — Yeh Kaise Kaam Karta Hai**:
Jab tum `@Transactional` method call karte ho, Spring asli object call nahi karta — woh ek **AOP proxy** (CGLIB ya JDK dynamic proxy) call karta hai. Proxy method entry pe `PlatformTransactionManager` se transaction shuru karta hai (DB connection le kar `BEGIN`), method body chalata hai, fir exception na ho to `COMMIT`, runtime exception aaye to `ROLLBACK`. Isi liye **same class ke andar se** `@Transactional` method ko self-call karo to proxy bypass ho jaata hai (transaction lagta hi nahi) — yeh classic interview gotcha hai.

Duplicate-row race ko solve karne ke liye hum DB-level **unique constraint `(cart_id, product_id)`** lagate hain. Do parallel INSERT mein se ek `DataIntegrityViolationException` khaayega — usko catch kar ke hum quantity update kar dete hain (ya `INSERT ... ON CONFLICT` use karte hain). DB ko hi truth ka guard banao, application memory ko nahi.

**Bhai, Simple Mein Samjho**:
`CartService` ek "cart ka manager" hai jiska ek hi kaam hai — cart sambhalna. Item add karo, quantity merge karo, stock se pucho "available hai?", bas. Pricing, email, analytics — yeh dusre logon ka kaam hai. Ek banda, ek zimmedari (high cohesion).

**Code Pattern**:
```java
@Service
@RequiredArgsConstructor
public class CartService {

    private final CartRepository cartRepo;
    private final CartItemRepository itemRepo;
    private final InventoryClient inventory;   // alag service — cohesion!

    @Transactional   // Spring AOP proxy yahaan transaction wrap karta hai
    public CartItemDto addToCart(Long userId, AddToCartRequest req) {
        // 1. Stock available? (inventory ka kaam inventory karta hai — delegate)
        int available = inventory.getAvailableStock(req.getProductId());
        if (available < req.getQuantity()) {
            throw new InsufficientStockException(req.getProductId(), available);
        }

        Cart cart = cartRepo.findByUserId(userId)
            .orElseGet(() -> cartRepo.save(new Cart(userId)));

        // 2. Upsert: pehle se hai to merge, warna naya
        return itemRepo.findByCartIdAndProductId(cart.getId(), req.getProductId())
            .map(existing -> {                       // merge quantity
                existing.setQuantity(existing.getQuantity() + req.getQuantity());
                return CartItemDto.from(existing);   // dirty checking auto-saves
            })
            .orElseGet(() -> {                       // naya item
                CartItem item = new CartItem(cart, req.getProductId(), req.getQuantity());
                return CartItemDto.from(itemRepo.save(item));
            });
    }
}
```

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {
    Optional<CartItem> findByCartIdAndProductId(Long cartId, Long productId);
}
```

**Interview phrasing**:
"Iss scenario mein main `CartService` ko highly cohesive rakhunga — sirf cart operations. Stock check `InventoryClient` ko delegate karunga. Duplicate race ke liye DB pe unique constraint `(cart_id, product_id)` lagaunga aur upsert karunga, kyunki application-level check-then-act concurrent requests mein toot jaata hai. `@Transactional` se atomicity ensure hogi."

---

## 🔧 Stack 2: .NET / C# (Enterprise Backend Perspective)

**Concept needed**: EF Core change tracker + `SaveChangesAsync` atomicity + cohesive `CartService` + DB unique index for upsert.

**Under the Hood — .NET Mein Yeh Kaise Kaam Karta Hai**:
EF Core ka **Change Tracker** har loaded entity ka snapshot rakhta hai. Jab tum `existing.Quantity += qty` karte ho, EF entity ko `Modified` state mein daal deta hai. `SaveChangesAsync()` pe woh sirf changed columns ka `UPDATE` generate karta hai, sab kuch ek hi implicit transaction mein wrap karta hai — agar koi statement fail ho to pura rollback. `AsNoTracking()` yahaan use **nahi** karenge kyunki hum write kar rahe hain (tracking chahiye dirty detection ke liye).

`async/await` compiler ek state machine bana deta hai — DB I/O pe await karte waqt thread pool ka thread free ho jaata hai dusre requests serve karne ke liye. Yeh high-concurrency flash sale mein critical hai.

**Bhai, .NET Mein Yeh Kaise Hota Hai**:
Same philosophy: `CartService` sirf cart sambhale. EF Core khud quantity change track karega, tumhe explicit `UPDATE` likhne ki zaroorat nahi — `SaveChangesAsync` sambhal lega.

**Code Pattern**:
```csharp
public class CartService
{
    private readonly AppDbContext _db;
    private readonly IInventoryClient _inventory;   // delegate — cohesion

    public CartService(AppDbContext db, IInventoryClient inventory)
        => (_db, _inventory) = (db, inventory);

    public async Task<CartItemDto> AddToCartAsync(long userId, AddToCartRequest req)
    {
        var available = await _inventory.GetAvailableStockAsync(req.ProductId);
        if (available < req.Quantity)
            throw new InsufficientStockException(req.ProductId, available);

        var cart = await _db.Carts.Include(c => c.Items)
                       .FirstOrDefaultAsync(c => c.UserId == userId)
                   ?? _db.Carts.Add(new Cart(userId)).Entity;

        var existing = cart.Items.FirstOrDefault(i => i.ProductId == req.ProductId);
        if (existing is not null)
            existing.Quantity += req.Quantity;          // change tracker marks Modified
        else
            cart.Items.Add(new CartItem(req.ProductId, req.Quantity));

        await _db.SaveChangesAsync();                   // atomic upsert
        return CartItemDto.From(cart, req.ProductId);
    }
}
```

**Java vs .NET Comparison Table**:
| Feature | Java/Spring | .NET/C# |
|---------|-------------|---------|
| Transaction | `@Transactional` (AOP proxy) | implicit `SaveChangesAsync` / `BeginTransactionAsync` |
| Dirty tracking | Hibernate persistence context | EF Core Change Tracker |
| Duplicate guard | `DataIntegrityViolationException` | `DbUpdateException` (unique index) |
| DI | `@RequiredArgsConstructor` | constructor injection |
| Async | `CompletableFuture` / virtual threads | `async/await` Task |

---

## 🔧 Stack 3: SQL (Data Layer)

**Concept needed**: Unique constraint `(cart_id, product_id)` + atomic **UPSERT** (`MERGE` / `ON DUPLICATE KEY UPDATE` / `ON CONFLICT`) to kill the duplicate-row race.

**Under the Hood — SQL Engine Yeh Kaise Karta Hai**:
Unique index ek B-tree hai jis pe DB engine har INSERT/UPDATE se pehle uniqueness check karta hai — **lock manager** us key range pe ek lock leta hai. Do concurrent INSERT same `(cart_id, product_id)` pe aayein to ek serialize hoga, ek error khaayega. Yeh check **DB engine ke andar atomic** hai — application memory check ki tarah race nahi hota.

MySQL ka `INSERT ... ON DUPLICATE KEY UPDATE` ek single atomic statement hai — agar unique key clash ho to insert ki jagah update kar deta hai, sab kuch ek lock ke andar. SQL Server ka `MERGE` similar hai (lekin `MERGE` ke kuch concurrency bugs mashhoor hain — production mein `MERGE` ke saath `HOLDLOCK` lagate hain ya simple `UPDATE`-then-`INSERT` pattern use karte hain).

**Bhai, Database Mein Yeh Problem Kaise Solve Hoti Hai**:
Application mein "pehle dekho phir daalo" do bandon ke beech race kara deta hai. DB ko bolo "tu khud guarantee kar ke ek hi (cart, product) ka row banega". Unique constraint = bouncer jo duplicate ko gate pe hi rok deta hai.

**SQL Example**:
```sql
-- One-time: duplicate-proof constraint
ALTER TABLE cart_items
  ADD CONSTRAINT UQ_cart_product UNIQUE (cart_id, product_id);

-- ✅ MySQL: atomic upsert — race-proof
INSERT INTO cart_items (cart_id, product_id, quantity, created_at)
VALUES (@cartId, @productId, @qty, NOW())
ON DUPLICATE KEY UPDATE
  quantity = quantity + VALUES(quantity),   -- merge, overwrite nahi
  updated_at = NOW();

-- ✅ SQL Server: safe upsert with explicit lock to avoid race
BEGIN TRANSACTION;
UPDATE cart_items WITH (UPDLOCK, SERIALIZABLE)
   SET quantity = quantity + @qty, updated_at = SYSUTCDATETIME()
 WHERE cart_id = @cartId AND product_id = @productId;

IF @@ROWCOUNT = 0
    INSERT INTO cart_items (cart_id, product_id, quantity, created_at)
    VALUES (@cartId, @productId, @qty, SYSUTCDATETIME());
COMMIT TRANSACTION;
```

**The Gotcha**:
Bina unique constraint ke, do concurrent `addToCart` calls dono `INSERT` kar denge — cart mein same product 2 baar, totals galat. Aur agar tum `quantity = @qty` (overwrite) likho `quantity = quantity + @qty` ke bajaye, to second add pehla quantity uda dega — user 3 daal raha tha, 1 reh gaya.

**Isolation Level Choice**:
Upsert ke liye `READ COMMITTED` kaafi hai **agar** unique constraint + atomic upsert statement use kar rahe ho (constraint hi serialize kar dega). Agar tum manual check-then-act kar rahe ho to `SERIALIZABLE` ya explicit `UPDLOCK` chahiye — warna phantom insert race ho jaayega.

---

## 🔧 Stack 4: Angular (Frontend Layer)

**Concept needed**: Cart state as a single source of truth via `BehaviorSubject` (cohesive `CartStore`) + **optimistic UI** add-to-cart with rollback.

**Under the Hood — Angular Yeh Kaise Karta Hai**:
`BehaviorSubject` ek special RxJS Subject hai jo "current value" yaad rakhta hai — naya subscriber turant latest cart state pa jaata hai (cart badge instantly populate). Jab hum `cart$.next(newState)` karte hain, sab subscribers (navbar badge, cart drawer, total) ek saath update ho jaate hain. Angular Change Detection (Zone.js) HTTP/async event ke baad component tree check karta hai; `async` pipe khud subscribe/unsubscribe sambhalta hai — memory leak se bachata hai.

**Optimistic UI**: hum API response ka wait nahi karte — UI turant "+1" dikha deta hai (badge 0 → 1), background mein API call jaati hai. Fail ho to rollback (badge wapas 0). Yeh perceived performance huge boost deta hai — FoodPanda pe button dabate hi count badhta hai, lag nahi.

**Bhai, Frontend Pe Yeh Kaise Handle Karein**:
Cart state ek hi jagah rakho (`CartStore`), poori app usi se cart padhe. Button dabate hi count badha do (optimistic), phir API confirm kare. Galti ho to undo — user ko fast bhi laga aur galat data bhi nahi raha.

**Code Pattern**:
```typescript
@Injectable({ providedIn: 'root' })
export class CartStore {
  private readonly _cart$ = new BehaviorSubject<Cart>(EMPTY_CART);
  readonly cart$ = this._cart$.asObservable();
  readonly itemCount$ = this.cart$.pipe(
    map(c => c.items.reduce((sum, i) => sum + i.quantity, 0))
  );

  constructor(private api: CartApiService, private toast: ToastService) {}

  addToCart(product: Product, qty = 1): void {
    const snapshot = this._cart$.value;                 // rollback ke liye

    // 1. OPTIMISTIC: turant UI update
    this._cart$.next(this.mergeLocally(snapshot, product, qty));

    // 2. CONFIRM: server se pakka karo
    this.api.addToCart({ productId: product.id, quantity: qty }).pipe(
      catchError(err => {
        this._cart$.next(snapshot);                     // 3. ROLLBACK on fail
        this.toast.error(err.status === 409
          ? 'Stock khatam ho gaya 😔'
          : 'Cart update fail, dobara try karein');
        return EMPTY;
      })
    ).subscribe(serverCart => this._cart$.next(serverCart)); // server truth se sync
  }

  private mergeLocally(cart: Cart, p: Product, qty: number): Cart { /* immutable merge */ }
}
```

**UX Concern**:
Bina single `CartStore` ke — har component apni cart copy rakhega, badge aur drawer out-of-sync ho jaayenge (navbar "2" dikhaye, drawer "1"). Bina optimistic UI ke — har add pe 400ms spinner, user ko laga app slow hai. Bina rollback ke — stock khatam hone par bhi UI count badha dikhayega, jhoot bol raha hai user se.

---

## 🔧 Stack 5: System Design (Architecture View)

**Pattern needed**: **Cart as a bounded service** — guest cart (Redis/cookie) vs persistent cart (DB), merge-on-login, aur **stock reservation strategy** (reserve-at-checkout, not at-add).

**Under the Hood — Yeh Architecture Pattern Kaise Kaam Karta Hai**:
Cart ek high-write, low-durability-need workload hai (log carts banate/chhodte rehte hain). Isiliye:
- **Guest cart** → Redis (key = session/cookie ID, TTL 7-30 din). Fast, ephemeral, koi DB load nahi.
- **Logged-in cart** → SQL DB (durable, multi-device sync).
- **Merge-on-login** → login pe Redis cart DB cart mein merge (quantity add, dedupe by product).
- **Stock**: add-to-cart pe sirf **"availability snapshot"** dikhao (read-only). Actual **reservation checkout pe** karo, ek short TTL hold ke saath (e.g., 10 min). Yeh oversell aur false-sold-out dono se bachata hai.

**Bhai, Architecture Level Pe Yeh Kaise Design Karein**:
Cart ko alag service rakho jo sirf cart jaanta hai (high cohesion). Inventory alag service, pricing alag. Cart guest ke liye Redis mein, logged-in ke liye DB mein. Stock checkout pe reserve karo, add pe nahi — warna cart abandonment se inventory jhooti khali dikhegi.

**Architecture Diagram**:
```
┌─────────────┐        ┌───────────────────┐
│   Angular   │  HTTP  │   Cart Service    │   (high cohesion: sirf cart)
│  CartStore  │───────▶│  add / merge /    │
│ optimistic  │        │  remove / view    │
│  itemCount$ │◀───────│                   │
└─────────────┘  JSON  └───────────────────┘
                          │            │
            guest cart    │            │   logged-in cart
                          ▼            ▼
                   ┌──────────┐   ┌──────────────┐
                   │  Redis   │   │  SQL DB      │
                   │ TTL 7d   │   │ carts +      │
                   │ (guest)  │   │ cart_items   │
                   └──────────┘   │ UQ(cart,prod)│
                          │       └──────────────┘
            login event → │ merge guest → DB
                          ▼
                   ┌───────────────────┐      ┌───────────────────┐
                   │ Inventory Service │      │ Pricing Service   │
                   │ availability read │      │ tax/discount calc │
                   │ reserve@checkout  │      │ (separate concern)│
                   └───────────────────┘      └───────────────────┘
```

**Trade-offs**:
| Approach | Pros | Cons |
|----------|------|------|
| Reserve stock at add-to-cart | No oversell, simple checkout | False "sold out" (70% carts abandon), inventory locked uselessly |
| Reserve at checkout (TTL hold) | Real availability, less lock contention | Slim oversell window — handle at payment with compensation |
| Guest cart in cookie only | Zero server state | Size limit (4KB), no cross-device, lost on cache clear |
| Guest cart in Redis | Fast, larger, server-controlled TTL | Extra infra, needs session ID |

**Real Companies Using This**:
- **Amazon**: Cart service decoupled; reservation at checkout with a hold timer ("items in your cart may sell out").
- **FoodPanda/Daraz**: Redis guest cart → merge on login → DB persistent cart.
- **Shopify**: Cart, Inventory, Pricing alag bounded contexts — textbook high cohesion at service level.

---

---

## 🏛️ OOP + DESIGN PATTERN LENS (Cross-Questioning Proof)

**Today's OOP/Pattern Focus**: **Cohesion — High Cohesion Principle** (Day 7 of OOP curriculum)

### Principle/Pattern Definition

**Concept**: Cohesion = ek class/module ke andar ke elements (fields + methods) ek doosre se kitne **related** hain, ek hi clear purpose ke around kitne tightly focused hain. **High cohesion** = class ka har method ek hi well-defined responsibility serve karta hai (`CartService` sirf cart). **Low cohesion** = class mein bekaar mix — cart + email + payment + reporting sab ek jagah (God Class).

> **Yaad rakho**: Cohesion = "andar ka focus" (ek class kitni focused hai). Coupling (kal ka topic) = "bahar ka dependency" (classes ek doosre pe kitni depend karti hain). **Goal: High cohesion + Loose coupling.**

**Bhai, Simple Mein Samjho**:
High cohesion = ek achha employee jiska ek hi clear job hai — accountant sirf hisaab kitaab kare. Low cohesion = woh banda jo accountant bhi hai, driver bhi, chowkidar bhi, cook bhi — sab kuch karta hai, kuch bhi theek se nahi. Confuse, untestable, replace karna mushkil.

**Real-Life Analogy (Pakistani Context)**:
Bhai, **biryani ki degh** socho. Achhe restaurant mein har banda specialized hai — **ek banda sirf biryani banata** (high cohesion), ek sirf karahi, ek sirf naan. Sab apne kaam ke master.

Ab socho ek dhaba jahan **ek hi banda** order leta hai, biryani banata hai, paise bhi leta hai, bartan bhi dhota hai, aur delivery bhi karta hai (low cohesion). Rush time pe sab kuch atak jaata hai, ek banda bemaar to poora dhaba band. Software mein bhi aisa "God Class" — ek change pe sab kuch toot-ne ka dar.

Yeh hi `CartService` ke saath hota hai — agar woh cart + pricing + inventory + email sab kare, to email logic badalne pe cart test bhi torna padta hai. **Ek class, ek wajah badalne ki.**

---

### Aaj Ke Brainer Mein Yeh Pattern/Principle Kahan Hai?

Aaj ke "Add to Cart" mein cohesion 3 jagahon pe decision banata hai:

1. **`CartService` ka scope**: Sirf cart operations — add, merge, remove, view, clear. Stock check `InventoryClient` ko delegate, pricing `PricingService` ko. **High cohesion.**

2. **Angular `CartStore`**: Sirf cart state manage karta hai (`BehaviorSubject`, item count, optimistic merge). HTTP `CartApiService` mein, toast `ToastService` mein. Har cheez apni jagah.

3. **❌ Anti-pattern jo log karte hain**: `CartService.addToCart()` ke andar — stock decrement, email confirmation, loyalty points add, analytics event, recommendation update — sab thoonk dena. Yeh **God Service**, low cohesion. 5 alag wajah se yeh ek method badalna padega.

---

### Code Mein Yeh Principle Kaise Apply Hota Hai?

**❌ BAD (Low Cohesion — God Service jo sab karta hai)**:
```java
@Service
public class CartService {

    public void addToCart(Long userId, Long productId, int qty) {
        // 1. Cart logic — theek hai
        CartItem item = cartItemRepo.upsert(userId, productId, qty);

        // 2. ❌ Inventory decrement — yeh cart ka kaam nahi
        Product p = productRepo.findById(productId);
        p.setStock(p.getStock() - qty);
        productRepo.save(p);

        // 3. ❌ Pricing/tax calculation — pricing ka kaam
        BigDecimal total = item.getPrice().multiply(BigDecimal.valueOf(qty));
        BigDecimal tax = total.multiply(new BigDecimal("0.17"));  // magic number bhi!

        // 4. ❌ Email — notification ka kaam
        emailSender.send(userRepo.findById(userId).getEmail(),
                         "Item added!", "...");

        // 5. ❌ Analytics — analytics ka kaam
        analytics.track("add_to_cart", productId);
    }
}
// Problems:
// - 5 alag wajah se yeh class badlegi (SRP bhi violate)
// - Email server down? Cart add fail ho jaayega (galat coupling)
// - Test karne ke liye email + analytics + inventory sab mock karne padenge
// - Naya dev confuse: "cart class email kyun bhej rahi hai?"
```

**✅ GOOD (High Cohesion — har class apna kaam)**:
```java
@Service
@RequiredArgsConstructor
public class CartService {                 // SIRF cart — focused
    private final CartItemRepository itemRepo;
    private final InventoryClient inventory;
    private final ApplicationEventPublisher events;

    @Transactional
    public CartItemDto addToCart(Long userId, AddToCartRequest req) {
        if (inventory.getAvailableStock(req.getProductId()) < req.getQuantity())
            throw new InsufficientStockException(req.getProductId());

        CartItem item = upsertItem(userId, req);          // core cart kaam

        // Side-effects ko event se decouple karo — fire & forget
        events.publishEvent(new ItemAddedToCart(userId, req.getProductId(), req.getQuantity()));
        return CartItemDto.from(item);
    }
}

// Alag cohesive listeners — har ek apni responsibility
@Component @RequiredArgsConstructor
class CartAnalyticsListener {
    @EventListener void onAdd(ItemAddedToCart e) { analytics.track("add_to_cart", e.productId()); }
}

@Component @RequiredArgsConstructor
class CartNotificationListener {
    @EventListener void onAdd(ItemAddedToCart e) { /* optional email/push */ }
}
// Wins:
// - CartService sirf cart ke liye badlegi
// - Email down? Cart phir bhi add hoga (listener async/isolated)
// - Test simple: bas cart logic test karo
// - Naya feature (loyalty points)? Naya listener add karo, CartService untouched
```

---

### Java vs .NET Mein Yeh Principle Kaise Implement Hota Hai?

| Aspect | Java | C#/.NET |
|--------|------|---------|
| Decoupled side-effects | `ApplicationEventPublisher` + `@EventListener` | `MediatR` notifications / `IPublisher` |
| Cohesive boundary | Package per feature (`cart`, `inventory`) | Folder/namespace + Feature folders |
| Single responsibility helper | private methods within service | private methods / local functions |
| Cross-feature contract | interface (`InventoryClient`) | interface (`IInventoryClient`) |
| Enforce boundaries | ArchUnit tests | NetArchTest / `.editorconfig` rules |

```csharp
// C# equivalent — cohesive cart service, side-effects via MediatR
public class CartService
{
    private readonly ICartItemRepository _items;
    private readonly IInventoryClient _inventory;
    private readonly IPublisher _publisher;            // MediatR

    public CartService(ICartItemRepository items, IInventoryClient inventory, IPublisher publisher)
        => (_items, _inventory, _publisher) = (items, inventory, publisher);

    public async Task<CartItemDto> AddToCartAsync(long userId, AddToCartRequest req)
    {
        if (await _inventory.GetAvailableStockAsync(req.ProductId) < req.Quantity)
            throw new InsufficientStockException(req.ProductId);

        var item = await UpsertItemAsync(userId, req);          // core cart kaam
        await _publisher.Publish(new ItemAddedToCart(userId, req.ProductId, req.Quantity));
        return CartItemDto.From(item);
    }
}

// Separate cohesive handlers
public class CartAnalyticsHandler : INotificationHandler<ItemAddedToCart> { /* track */ }
public class CartNotificationHandler : INotificationHandler<ItemAddedToCart> { /* notify */ }
```

---

### 🎯 Cross-Questioning Drill (Interview Defense)

**Cross Q1**: "High cohesion ka faida kya? Sab kuch ek hi CartService mein rakhne se to kam files, kam navigation — simple lagta hai?"
**Confident Answer**:
"Short term mein ek file simple lagti hai — agreed. Lekin high cohesion 3 concrete faide deti hai. **(1) Change isolation**: agar `CartService` sirf cart jaanti hai, to email template badalna ho to main `CartService` ko haath nahi lagata — `NotificationListener` change karta hoon. God class mein email change karne pe galti se cart logic toot sakti hai, aur cart ke tests bhi dobara chalane padte hain. **(2) Testability**: cohesive class ke kam dependencies hote hain — `CartService` test karne ke liye sirf repo + inventory mock chahiye, email/analytics/payment nahi. **(3) Cognitive load**: naya developer `CartService` khole to use saaf pata chale yeh sirf cart ka kaam karti hai. God class 1000 lines ki ho to samajhne mein ghanta lagta hai. Cohesion = code ka 'ek nazar mein samajh aana'."

---

**Cross Q2**: "Cohesion aur SRP (Single Responsibility) mein kya farq hai? Same cheez nahi?"
**Confident Answer**:
"Bahut related hain, par same nahi. **Cohesion** ek **measure/property** hai — kitni focused hai class (high ya low, ek spectrum). **SRP** ek **rule/guideline** hai jo kehta hai 'ek class ki badalne ki sirf ek wajah honi chahiye'. SRP follow karoge to high cohesion automatically aa jaayegi — SRP ko achieve karne ka tareeqa high cohesion design karna hai. Cohesion purana concept hai (Larry Constantine, 1970s, structured design se), SRP usi ko Robert Martin ne OOP/SOLID context mein refine kiya. Interview phrasing: 'High cohesion is the property; SRP is the principle that drives you toward it.' Aur cohesion ke types bhi hain — functional cohesion (best, sab ek task ke liye), down to coincidental cohesion (worst, random cheezein ek jagah)."

---

**Cross Q3**: "Yeh principle Spring/EF Core mein automatically aata hai ya manual design?"
**Confident Answer**:
"Manual design hai — framework cohesion enforce nahi karta. Spring tumhe khushi se 2000-line `@Service` chalane dega jo sab kuch kare. Lekin frameworks **tools** dete hain cohesion maintain karne ke liye: Spring ka `ApplicationEventPublisher` aur `@EventListener` se side-effects decouple karo, `@Component` scanning se feature-wise packages banao. .NET mein MediatR isi liye popular hai. Aur enforcement ke liye **ArchUnit (Java)** ya **NetArchTest (.NET)** se architecture tests likho — e.g., 'cart package inventory package ke internal classes import na kare'. Yeh CI mein cohesion/coupling rules ko code bana deta hai. Bottom line: discipline tumhari, tooling framework ki."

---

**Cross Q4**: "High cohesion ko bhi over-do kiya ja sakta hai? Kabhi nuksan?"
**Confident Answer**:
"Bilkul — extreme cohesion ka result hai **over-fragmentation / anemic classes**. Agar har chhoti cheez ke liye alag class bana do — `AddToCartHandler`, `RemoveFromCartHandler`, `MergeCartHandler`, `ClearCartHandler` — to ek simple cart flow samajhne ke liye 10 files khole padte hain. Yeh 'class explosion' anti-pattern hai. **Balance**: related operations ek cohesive class mein rakho (`CartService` ke andar add/remove/merge theek hai — sab cart hi to hai). Alag tab karo jab responsibility genuinely alag ho (inventory, pricing). Aur ek aur trap: **anemic domain model** — entity sirf getters/setters, saara logic service mein — yeh 'data' aur 'behavior' ko alag kar deta hai, jo cohesion ke against hai (behavior data ke saath hona chahiye). So cohesion ka matlab 'related cheezein saath, unrelated alag' — dono direction mein balance."

---

**Cross Q5**: "Production mein, microservices mein cohesion kaise scale karta hai?"
**Confident Answer**:
"Microservices mein cohesion **service boundary** define karta hai. DDD ka **Bounded Context** essentially 'high cohesion at service level' hai — Cart Service sirf cart ka domain own kare, Inventory Service sirf stock, Pricing sirf paisa. Galat boundary (low cohesion service) = ek business change pe 3 services deploy karne padein — distributed monolith ban jaata hai (worst of both worlds). Sahi cohesion = ek feature ek service mein, independently deployable. Test: 'agar pricing rule badle, kitne services deploy hote?' Sahi design mein sirf Pricing Service. Amazon/Netflix ka rule: 'two-pizza team owns one cohesive service'. Cohesion organizational level pe bhi jaata hai (Conway's Law) — team structure service boundaries ko mirror karti hai."

---

### 🧩 Pattern Combination (How Today's Pattern Pairs With Others)

**Pairs Well With**:
- **Single Responsibility Principle (SRP)** — Cohesion SRP ka measurable form hai; SRP follow karo, high cohesion milegi.
- **Loose Coupling (kal ka topic)** — Best design = high cohesion + loose coupling. Cohesive cart service jo interface (loose coupling) se inventory se baat kare.
- **Observer / Pub-Sub (events)** — Side-effects ko event listeners mein nikaalo taake core service cohesive rahe.
- **Facade Pattern** — Ek cohesive facade jo complex subsystem ko simple interface deta hai.

**Conflicts With**:
- **Over-fragmentation / class explosion** — Cohesion ke naam pe har micro-operation alag class banana ulta nuksan.
- **God Object anti-pattern** — Direct opposite; low cohesion ka extreme.
- **Anemic domain model** — Behavior ko data se kaat ke alag service mein daalna cohesion ke against (behavior + data saath hona chahiye).

---

### 🎓 Real Production Code Where This Matters

Spring Framework khud high cohesion ka master example hai:
```java
// Har class ek hi clear cheez sambhalti hai
RestTemplate     // sirf HTTP calls
JdbcTemplate     // sirf JDBC operations
JpaRepository    // sirf data access
TransactionTemplate // sirf transaction boundaries
```
`RestTemplate` mein database logic nahi milega, `JdbcTemplate` HTTP nahi karta. **Har class ek razor-sharp responsibility** — isi liye Spring 20 saal se maintainable hai aur log inko alag-alag samajh sakte hain.

.NET mein bhi same: `HttpClient` (HTTP), `DbContext` (data), `ILogger` (logging), `IMemoryCache` (caching) — har ek tightly focused. Yeh "high cohesion at framework level" hai jo poore ecosystem ko learnable banata hai.

Real-world failure: monolithic `OrderManager` classes jo order + payment + shipping + email + inventory sab karti hain — yeh legacy enterprise codebases ka #1 maintenance nightmare hai. Inhe refactor karne ke liye hi "Extract Class" refactoring exist karti hai.

---

### 💡 Memory Hook for This Principle/Pattern

**Cohesion Memory Hook**: **"EK BANDA, EK KAAM — BIRYANI WALA SIRF BIRYANI"**
- High cohesion = biryani wala sirf biryani banata, master apne kaam ka.
- Low cohesion = dhaba ka ek banda jo sab kuch karta, kuch bhi theek nahi.

Aur ek aur:
- **"COHESION = ANDAR JOR, COUPLING = BAHAR DOORI"** — High cohesion (andar elements jude), loose coupling (bahar classes door). Magnet ke andar atoms aligned, magnets ek doosre se attached nahi.

---

### ⚠️ Common Mistakes Jo Aaj Bachne Hain

**❌ Mistake 1**: Cohesion aur Coupling ko ulat samajhna.
**Why it's wrong**: Log keh dete hain "high coupling achhi hai" — galat. **High cohesion** achhi (andar focus), **low/loose coupling** achhi (bahar kam dependency). Confuse mat karo.
**Correct approach**: Mantra yaad rakho: "**High** cohesion, **Loose** coupling." Cohesion zyada chahiye, coupling kam.

**❌ Mistake 2**: Cohesion ke naam pe har method ko alag class banana (class explosion).
**Why it's wrong**: 8 cart operations = 8 handler classes = simple flow samajhne ko 8 files. Over-engineering, navigation hell.
**Correct approach**: **Related** operations ek cohesive class mein (cart add/remove/merge saath). Alag tab karo jab domain genuinely alag ho (cart vs inventory vs pricing).

**❌ Mistake 3**: Side-effects (email, analytics, logging) ko core business method mein thoonk dena.
**Why it's wrong**: Cart service email pe coupled ho jaati hai — email server down to cart add fail. Aur 5 wajah se class badalti hai.
**Correct approach**: Domain event publish karo (`ItemAddedToCart`), alag cohesive listeners side-effects sambhalein. Core service focused rehti hai.

---

## 🧠 The Complete Solution: How All 5 Stacks Interlock

**The Flow** (Top to Bottom):

```
1. USER ACTION (Angular)
   └─▶ "Add to Cart" click → CartStore optimistic +1 (badge instant)
       └─▶ background HTTP POST /api/cart/items

2. API REQUEST (Spring Boot / .NET)
   └─▶ CartController → CartService.addToCart() (@Transactional proxy)
       └─▶ cohesive: sirf cart, baaki delegate

3. BUSINESS LOGIC (Java / C#)
   └─▶ inventory.getAvailableStock() check (delegate — high cohesion)
       └─▶ upsert cart item (merge quantity if exists)
           └─▶ publish ItemAddedToCart event (decoupled side-effects)

4. DATABASE (SQL)
   └─▶ Unique constraint (cart_id, product_id) → atomic upsert
       └─▶ ON DUPLICATE KEY UPDATE quantity = quantity + @qty (race-proof)

5. ARCHITECTURE (System Design)
   └─▶ guest? Redis cart : DB cart  | stock reserved at CHECKOUT not add
       └─▶ analytics/notification listeners react async

6. RESPONSE
   └─▶ DB success → 200 + server cart → CartStore syncs (truth)
       └─▶ on 409 (stock) → rollback optimistic, toast "stock khatam"
```

**What Breaks If You Skip ANY Layer**:
- **Skip optimistic UI (Angular)**: Har add pe 400ms spinner — app slow lagega, conversion girega.
- **Skip cohesion (Backend)**: God CartService — email down to cart add fail, har change risky.
- **Skip unique constraint (SQL)**: Double-click → 2 duplicate rows → totals galat → user trust gone.
- **Skip reserve-at-checkout (Architecture)**: Add pe stock lock → 70% abandoned carts inventory ko jhooti "sold out" bana dein.
- **Skip merge-on-login**: Guest ne 5 items daale, login pe sab gayab — user furious.

---

## 🧭 MENTAL MAP — How to Memorize This

```
                  [ADD TO CART BRAINER]
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    [Frontend]       [Backend]          [Database]
        │                 │                 │
    Angular          Spring/.NET           SQL
        │                 │                 │
    CartStore        CartService        UQ(cart,product)
    BehaviorSubject  (cohesive!)        atomic UPSERT
    optimistic +1    delegate stock     quantity += qty
    rollback 409     publish event      READ COMMITTED
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                  [SYSTEM DESIGN]
                  Guest=Redis, User=DB
                  merge-on-login
                  reserve @ checkout
                          │
                  [OOP LENS: COHESION]
                  Ek class ek kaam
                  Biryani wala sirf biryani
```

**Mental Story to Remember (Roman Urdu)**:
Imagine karo tum FoodPanda restaurant ke owner ho aur "Add to Cart" team bana rahe ho:
- **Waiter (Angular CartStore)**: Customer ke bolte hi order slip pe "+1" likh deta hai (optimistic), kitchen ko confirm bhejta hai. Kitchen na bole "khatam", to slip se kaat deta hai (rollback).
- **Manager (CartService)**: Sirf order coordinate karta hai (cohesive) — khaana khud nahi banata, stock khud nahi check karta. Storekeeper se pucchta hai "available?", phir order book mein add karta hai.
- **Storekeeper (SQL)**: Order book mein same item do baar nahi likhne deta (unique constraint) — agar pehle se hai to quantity badha deta hai (upsert).
- **Restaurant System (Architecture)**: Guest ke liye temporary notepad (Redis), regular customer ke liye permanent file (DB). Ingredients tabhi reserve karta jab customer pakka order de (checkout), browse pe nahi.
- **Cohesion (OOP)**: Biryani wala sirf biryani, manager sirf manage. Koi confuse nahi, sab apne kaam ke master.

Agar koi bhi tooti — duplicate rows, slow UI, oversell, ya guest cart gum. Pura system inter-locked hai.

**Acronym/Mnemonic**: **"C-MORE"**
- **C** = Cohesion (cart service sirf cart)
- **M** = Merge (upsert quantity, duplicate na bane)
- **O** = Optimistic UI (frontend instant +1)
- **R** = Reserve at checkout (stock, not at add)
- **E** = Event for side-effects (decouple email/analytics)

Yaad rakho: "Cart ko **C-MORE** chahiye — duplicate kam, conversion zyada."

---

## 🎤 INTERVIEW DRILL SECTION

### Main Interview Question

**Q**: "Aap ek e-commerce ke liye 'Add to Cart' design karein. Requirements: duplicate product cart mein na bane (quantity merge ho), stock check ho, guest aur logged-in dono ka cart chale, aur flash sale mein 1000 concurrent adds handle hon. Kaise approach lenge?"

**Expected Answer (60-90 seconds)**:

**Opening** (10 sec):
"Main 5 layers coordinate karunga, aur design ka core principle hoga **high cohesion** — cart service sirf cart sambhale, stock aur pricing alag services ko delegate kare. Duplicate aur concurrency ko DB-level guarantees se solve karunga, application memory checks pe nahi."

**Body** (60 sec):
1. **Frontend (Angular)**: Single `CartStore` (`BehaviorSubject`) as source of truth — badge, drawer, total sab isi se. **Optimistic UI** — click pe instant +1, fail (409 stock) pe rollback + toast. Double-click debounce/disable button.
2. **Backend API (Spring/.NET)**: `CartController` → cohesive `CartService.addToCart()` with `@Transactional`. Stock `InventoryClient` se, side-effects (analytics/email) domain event se decouple.
3. **Business Logic (Java/C#)**: Upsert — agar item hai to quantity merge, warna naya. Stock insufficient pe `InsufficientStockException` → 409.
4. **Database (SQL)**: **Unique constraint `(cart_id, product_id)`** — duplicate-proof at DB level. **Atomic upsert** (`ON DUPLICATE KEY UPDATE quantity = quantity + ?`). `READ COMMITTED` kaafi hai constraint ke saath.
5. **Architecture**: Guest cart Redis (TTL), logged-in DB, **merge-on-login**. Stock **reserve at checkout** (short TTL hold), add-to-cart pe sirf availability snapshot — taake abandoned carts inventory na lock karein.

**Closing** (10 sec):
"High cohesion + DB-level uniqueness se hum guarantee karte hain: koi duplicate row nahi, oversell nahi, fast UI, aur kal koi naya side-effect (loyalty points) chahiye to ek naya event listener — `CartService` untouched."

### Under-the-Hood Concepts You MUST Know

1. **Check-then-act race**: `if (notExists) insert()` concurrent requests mein toot-ta hai — dono "not exists" dekh ke dono insert. Fix: DB unique constraint + atomic upsert.
2. **Spring `@Transactional` self-invocation**: Same class ke method ko `this.method()` se call karne pe AOP proxy bypass — transaction lagti hi nahi. Inject self ya alag bean use karo.
3. **EF Core Change Tracker**: `entity.Quantity += x` ko `Modified` mark karta hai, `SaveChangesAsync` sirf changed columns ka UPDATE banata, ek implicit transaction mein.
4. **BehaviorSubject vs Subject**: BehaviorSubject latest value cache karta hai — naya subscriber turant current cart pa jaata hai (badge instantly correct).
5. **Stock reservation timing**: reserve-at-add (no oversell, but false sold-out + lock waste) vs reserve-at-checkout (real availability, slim oversell window handled with compensation).

### 3 Counter-Questions Interviewer Will Ask (With Detailed Answers)

**Counter Q1 (Trade-off focused)**: "Stock add-to-cart pe reserve karein ya checkout pe? Dono ke trade-off batao."
**Your Answer**:
"**Reserve-at-add**: oversell zero (jaise hi cart mein, stock kam), lekin do bade nuksan — (1) cart abandonment 60-70% hota hai, to inventory artificially 'sold out' dikhega jabki asal mein available hai; (2) carts mein pada stock genuine buyers ko block karta hai. **Reserve-at-checkout**: user ko real availability dikhti hai, inventory efficiently use hoti hai, lekin ek slim oversell window — checkout aur payment ke beech 2 log last item pe ja sakte hain. Isko hum **short TTL hold** (e.g., 10 min reservation at checkout start) + **payment-time final check** se handle karte hain; rare oversell pe compensation (refund + apology + alternative). **Industry standard reserve-at-checkout hai** — Amazon, BookMyShow (seat ko 10 min hold), Daraz. High-contention scarce items (concert tickets) ke liye queue + reservation hold use hota hai. Default: reserve at checkout, conversion ke liye."

---

**Counter Q2 (Scale focused)**: "Flash sale — 11 baje 50,000 log ek hi product (100 stock) ek second mein cart mein daal rahe. Kya badlega?"
**Your Answer**:
"Pure-DB approach is spike pe choke kar dega — row-level lock contention par sab serialize ho jaayega. Layers: **(1) Add-to-cart ko stock se decouple** — cart mein daalna sirf cart write hai (Redis, super fast), oversell ka sawaal checkout pe aata hai. **(2) Inventory ke liye atomic counter** — Redis `DECR` ya distributed lock; ya better, ek **reservation queue** (Kafka) — requests serialize, pehle 100 ko 'reserved', baaki ko 'waitlist/sold out' turant. **(3) Rate limiting + queue** — 'You are in line' page (jaise ticket sales), backend ko stampede se bachao. **(4) Idempotency keys** — double-click/retry se double-reserve na ho. **(5) Cache product page** at CDN edge — DB ko read traffic se bachao. Monitoring: P99 latency, Redis hit ratio, reservation success rate. Key insight: **add-to-cart ko light rakho, contention ko checkout/reservation pe localize karo.**"

---

**Counter Q3 (Failure mode focused)**: "Add-to-cart pe stock check pass ho gaya, item cart mein add bhi ho gaya, lekin response user tak pohanchne se pehle network toot gaya. User dobara click karta hai. Kya hoga?"
**Your Answer**:
"Yeh classic **retry/idempotency** problem hai. Bina protection ke — user ka pehla request actually succeed hua tha (quantity +1), dobara click se phir +1 ho jaayega — galat quantity. **Solutions**: (1) **Idempotency key** — client har add-to-cart pe ek unique key (UUID) bheje; server dekhe key already processed to wahi result return kare, dobara apply na kare (Redis mein key→result short TTL). Stripe exactly yeh karta hai. (2) **Frontend**: button disable jab tak response na aaye, optimistic state se double-fire rok do. (3) **Server upsert idempotent design**: agar 'set quantity to N' semantics ho (increment ke bajaye) to retry safe — lekin add-to-cart usually increment hai, isliye idempotency key zaroori. (4) **Graceful UI**: timeout pe 'confirming...' dikhao, cart refresh kar ke actual state dikhao na ki blind retry. **Fail safe: idempotency key + button disable + state reconciliation.**"

### Bonus: "Senior Engineer Differentiator" Question

**Q**: "Tumhari `CartService` ka `addToCart` method dheere-dheere bada hota ja raha hai — naye requirements (coupon, gift wrap, loyalty points, notification) aate rehte hain. Kaise manage karoge ke yeh God Class na bane?"
**Junior Answer**: "Main har naya feature ek private method mein daal dunga `addToCart` ke andar — saaf rahega. Ya alag `if` conditions add kar dunga features ke liye."
**Senior Answer**: "Private methods sirf cosmetic fix hain — class abhi bhi 5 wajah se badalti hai (low cohesion). Main **strategy yeh lunga**: **(1)** Core `addToCart` ko cohesive rakhun — sirf cart item upsert + stock guard. **(2)** Side-effects (loyalty, notification, analytics) ko **domain events** (`ItemAddedToCart`) se decouple karun — har concern ka apna cohesive listener, alag deploy/test ho sake. **(3)** Cross-cutting cart-modifications (coupon, gift wrap) ko **separate cohesive services** mein — `PricingService`, `PromotionService` — jinhe cart **delegate** kare. **(4)** Architecture tests (ArchUnit/NetArchTest) likhun jo enforce karein 'CartService notification package import na kare' — cohesion ko CI mein lock kar dun. **(5)** Periodically 'reason to change' audit — agar ek class 3+ wajah se badal rahi hai, **Extract Class** refactor. Yeh SRP + cohesion + Open/Closed ka combination hai — naya feature add karna ek naya listener/service likhna ho, existing code modify karna nahi."

### Red Flag Signals (Don't Say These!)

- ❌ "Pehle `SELECT` kar ke check karunga item hai ya nahi, phir `INSERT` ya `UPDATE`" — Why: Check-then-act race condition, concurrent requests pe duplicate rows. DB-level atomicity miss.
- ❌ "Add-to-cart pe hi stock minus kar dunga" — Why: Cart abandonment se inventory jhooti khali, genuine buyers blocked. Production mein conversion killer.
- ❌ "Sab cart-related cheezein (email, points, analytics) `CartService` mein daal dunga, ek hi jagah convenient hai" — Why: Low cohesion / God class, har change risky aur untestable.
- ❌ "Guest cart ki zaroorat nahi, login karwa lo pehle" — Why: Forced login conversion 30%+ girata hai; guest cart industry standard hai.
- ❌ "Frontend pe wait karwa do response ka, optimistic UI complex hai" — Why: Har add pe spinner = slow perceived performance = lower conversion.

---

## 📊 What You Should Be Able to Explain After Today

1. ✅ Check-then-act race condition kya hai aur DB unique constraint + atomic upsert kaise duplicate cart rows rokta hai.
2. ✅ High cohesion kya hai, low cohesion (God Class) se kya farq, aur cohesion vs coupling vs SRP ka rishta.
3. ✅ Stock ko add-to-cart pe reserve karein ya checkout pe — trade-offs aur industry standard.
4. ✅ Optimistic UI with rollback Angular mein `BehaviorSubject` ke saath kaise implement hoti hai.
5. ✅ Guest cart (Redis) vs persistent cart (DB) aur merge-on-login architecture.

---

## 🔗 Tomorrow's Connection Hint

Tomorrow we'll cover: **Day 8 — View Cart with Totals**

Aaj ke concept se kaise connected hai:
Aaj humne items cart mein **daale**. Kal woh cart **dikhana** hai with subtotal, tax, discount, grand total — yahaan se pricing logic, decimal precision (`BigDecimal`/`decimal`, kabhi `double` nahi paison ke liye!), aur OOP overlay **Interface vs Abstract Class** start hoga (e.g., `DiscountStrategy` interface vs `AbstractTaxCalculator`). Aaj ki cohesion lesson kal kaam aayegi — pricing alag cohesive service, cart se delegate.

---

## 📚 Progress Tracker

```
🟢 Beginner     ▓▓▓▓▓▓▓░░░░░░░░░░░░░ Day 7/20
🟡 Intermediate ░░░░░░░░░░░░░░░░░░░░ Day 0/30
🟠 Advanced     ░░░░░░░░░░░░░░░░░░░░ Day 0/40
🔴 Expert       ░░░░░░░░░░░░░░░░░░░░ Day 0/50
⚫ Master       ░░░░░░░░░░░░░░░░░░░░ Day 0/60
🟣 Architect    ░░░░░░░░░░░░░░░░░░░░ Day 0/70
💎 Principal    ░░░░░░░░░░░░░░░░░░░░ Day 0/95

Overall: 7/365 (1.9%)
```
