# 🔑 Day 12 — Upload Profile Picture: Revision Keys

**📖 Full lesson**: [2026-05-24-day12-upload-profile-picture.md](../lessons/01-beginner/2026-05-24-day12-upload-profile-picture.md)
**⏱️ Reading time**: 5-7 minutes
**🎯 Use when**: Night before interview, weekly revision, concept refresh

---

## ⚡ If You Remember Only 10 Things

1. **SNAP** = **S**ignature+Size validate (magic bytes, not Content-Type) + **N**o local disk (object storage, DB stores URL) + **A**sync transform (resize + EXIF strip) + **P**ublish via CDN with cache-busted URL.
2. **NEVER trust `Content-Type` / file extension** — client controls them (spoofable → RCE). Detect real type via **magic bytes** (Apache Tika / `Files.probeContentType`). JPEG=`FF D8 FF`, PNG=`89 50 4E 47`.
3. **Don't store binary in DB or local disk.** Object storage (S3/Blob) + DB stores only `avatar_key` + `version`. Local disk breaks on multi-server; DB BLOB pollutes buffer pool + bloats backups.
4. **EXIF strip = privacy + security.** EXIF carries **GPS location** of the photo. Re-encoding (decode → rewrite as new `BufferedImage`) kills all metadata + embedded payloads.
5. **Zip/decompression bomb guard**: check dimensions BEFORE full decode + hard size limit (`max-file-size=5MB`). A `60000×60000` PNG = ~14GB alloc → OOM/DoS.
6. **Async processing** (resize/thumbnail/WebP) off the request thread via `@Async`+`@EventListener` (Java) / `Channel<T>`+`BackgroundService` (.NET). Return **202 Accepted** with "PROCESSING".
7. **Cache-busting**: avatar URL must carry a **version** (`?v=4` or `/v4/`). Same URL = browser/CDN serves stale → "upload kaam nahi karta" bug.
8. **Orphan cleanup order**: DB **commit first**, **then** delete old file (background, idempotent). Reverse order + commit fail = file gone, DB says it exists = broken image forever.
9. **OCP** ("EXTEND HAAN, MODIFY NAA") = open for extension, closed for modification. New storage provider / transform = **new class** implementing an interface; `ProfilePictureService` untouched. Replaces `if/else` ladder.
10. **Presigned URL** (advanced): browser uploads **directly** to S3 with a temporary signed URL; backend only signs (ms) → offloads app-server bandwidth at scale.

---

## ☕ Java / Spring Boot — Quick Keys

- **`MultipartFile`** = wrapper over uploaded file: `getInputStream()`, `getSize()`, `getOriginalFilename()`, `getContentType()`. **Filename + ContentType come from client — untrusted.**
- **Multipart parsing**: `StandardServletMultipartResolver` splits body on boundary; `file-size-threshold` → small in memory, large spills to temp disk.
- **Apache Tika** `tika.detect(stream)` reads leading bytes → returns real MIME (magic-byte detection). Use a markable/resettable stream so you can re-read for storage.
- **Streaming upload**: `RequestBody.fromInputStream(in, size)` (AWS SDK v2) — no full-file RAM load. Avoid `file.getBytes()` (loads whole file).
- **`@Async`** = Spring AOP proxy submits to a `TaskExecutor` thread pool, returns immediately. Use a **dedicated pool** (bulkhead) for image work.
- **`@EventListener` + `ApplicationEventPublisher`** = decouple upload (fast) from processing (heavy/async).
- **OCP wiring**: inject `List<StorageProvider>` / `List<ImageTransform>` / `List<UploadValidator>` — Spring injects ALL impls. Choose active provider via `@Primary`/`@Qualifier`. Order pipeline with `@Order`.
- **Resize**: `BufferedImage` + `Graphics2D.drawImage` with `VALUE_INTERPOLATION_BILINEAR`. New `BufferedImage` = no EXIF carried over.
- **Size limits**: `spring.servlet.multipart.max-file-size` / `max-request-size` (defence-in-depth DoS guard).

---

## 🌐 .NET / C# — Quick Keys

- **`IFormFile`** = .NET's uploaded file. `OpenReadStream()`, `Length`, `FileName`, `ContentType` (client-controlled — untrusted).
- **Large files**: default model binding buffers; use `MultipartReader` to stream manually for big uploads.
- **Size limit**: `[RequestSizeLimit(5_000_000)]` / `MultipartBodyLengthLimit` / Kestrel `MaxRequestBodySize`.
- **Storage SDKs**: `AWSSDK.S3` `TransferUtility` or Azure `BlobClient.UploadAsync(stream, headers)`.
- **Background processing**: `Channel<T>` (producer/consumer queue) + `BackgroundService` (long-running hosted service). Equivalent to Java `@Async` listener.
- **Image lib**: `System.Drawing.Common` is **unsupported on Linux** — use **ImageSharp** or **SkiaSharp** (cross-platform).
- **OCP wiring**: inject `IEnumerable<IStorageProvider>` / `IEnumerable<IImageTransform>`; register via `AddSingleton<I, Impl>()` or keyed services for named resolution.

---

## 🗄️ SQL — Quick Keys

- **Don't BLOB**: SQL stores 8KB pages; large binaries go to off-row LOB pages → buffer-pool pollution (hot rows evicted), backup bloat (1M×2MB=2TB), replication lag, no CDN.
- **Store pointer**: `avatar_key VARCHAR(255)`, `avatar_version INT`, `avatar_mime`, `avatar_bytes`, `avatar_updated_at DATETIME2`.
- **Atomic version bump**: `UPDATE users SET avatar_key=@k, avatar_version=avatar_version+1, ... OUTPUT inserted.avatar_version WHERE id=@userId`. `OUTPUT` returns new value in one statement (race-safe).
- **Ownership**: always `WHERE id=@userId` (logged-in user) — prevents IDOR.
- **Orphan tracking**: `image_asset(owner_user_id, storage_key, status['ACTIVE'|'ORPHAN'|'DELETED'], created_at)` with index `(status, created_at)`. Mark old ORPHAN + insert new ACTIVE in one txn; delete files AFTER commit.
- **Isolation**: READ COMMITTED is enough — atomic UPDATE takes row X-lock, double-uploads serialize.

---

## 🎨 Angular — Quick Keys

- **Upload progress**: `http.post(url, formData, { reportProgress: true, observe: 'events' })`. Switch on `HttpEventType.UploadProgress` (`event.loaded`/`event.total`) and `HttpEventType.Response`.
- Under the hood: Angular subscribes XHR `upload.onprogress` via `HttpXhrBackend`; emits `HttpUploadProgress` events.
- **`FormData`**: `form.append('file', file, file.name)` → sent as `multipart/form-data`.
- **Instant preview**: `URL.createObjectURL(file)` makes a temporary blob URL — preview without uploading.
- **Memory leak guard**: `URL.revokeObjectURL(url)` on replace and in `ngOnDestroy` — blob URLs are held until revoked.
- **Client-side validation** (size/type) = UX only, NOT security (backend re-validates).
- **Cache-busting on result**: `this.avatar = url + '?v=' + version` so the new DP shows immediately.
- **`event.total` can be undefined** — guard before dividing.

---

## 🏗️ System Design — Quick Keys

- **Proxy-through-backend** (simple): browser→app→S3. App server bandwidth/memory bound.
- **Presigned URL direct upload** (scalable): app signs a temp URL (ms), browser PUTs directly to S3, S3 `ObjectCreated` event triggers processing. Offloads app servers.
- **Async pipeline**: upload → store original → DB pointer → queue (SQS/Kafka) → worker resizes + strips EXIF + WebP + thumbnails → CDN invalidate.
- **CDN (CloudFront)**: serve images at edge, milliseconds, offload origin. Combine with cache-busted versioned URLs.
- **Idempotent workers**: queue is at-least-once → same version key overwrite; N retries → **DLQ**.
- **On-the-fly transforms** (Cloudinary/imgix) at huge scale: store 1 original, generate sizes lazily via URL params, cache at CDN.
- **Real companies**: Instagram (multi-size + CDN), WhatsApp (EXIF strip + compress), Cloudinary/imgix (on-the-fly), Daraz/FoodPanda (S3 + CloudFront).

---

## 🏛️ OOP — Open/Closed Principle (OCP) Quick Keys

- **Definition**: open for **extension**, closed for **modification**. New behavior = new code (new class), not editing tested code. (Bertrand Meyer, 1988.)
- **Mnemonic**: **"EXTEND HAAN, MODIFY NAA"** + **"USB-C PORT"** — plug new device (extend), don't open the phone (modify). Port = interface.
- **Mechanism**: polymorphism — put what varies behind an interface; high-level code depends on interface; new variation = new impl, zero existing change.
- **Achieved via**: Strategy + DI (`List<Interface>` / `IEnumerable<Interface>`).
- **Pairs with**: Strategy (Day 62), Factory (Day 32), DI (Day 37), Chain of Responsibility (Day 64), Decorator (Day 44), Template Method (Day 66).
- **Tension with**: YAGNI/KISS — one impl doesn't need an interface. Rule of Three: concrete first, abstract on the **second** variation.
- **Real OCP in the wild**: `javax.imageio.ImageIO` SPI plugins (add format via `ServiceLoader`, ImageIO untouched), Spring `List<HttpMessageConverter>`, Logback appenders, Java `Comparator`, .NET middleware pipeline.
- **Violation smell**: `if (type == "s3") ... else if (type == "azure")` ladder — every new type edits tested code.
- **Common mistakes**:
  1. `if/else`/`switch` ladder on a type string → polymorphic dispatch instead.
  2. Constantly adding methods to the interface → breaks all impls (also ISP violation); keep interface stable.
  3. Over-abstraction for a single impl → premature; YAGNI.

---

## 🎤 Interview One-Liners (Bolne Layak Confidently)

- *"Profile picture upload mein main client ke Content-Type pe bharosa nahi karta — Apache Tika se magic bytes detect karta hun, kyunki extension/header spoofable hai aur RCE ka rasta hai."*
- *"File DB ya local disk pe nahi — S3 pe streaming, DB mein sirf key + version. App servers stateless rehte hain, aur CDN se direct serve hoti hai."*
- *"Resize aur EXIF strip async event listener mein hota hai — request thread block nahi hota, main 202 Accepted return karta hun. EXIF isliye strip karta hun kyunki usme user ki GPS location hoti hai."*
- *"Cache-busting ke liye URL mein version daalta hun — warna CDN purani DP serve karta rahega."*
- *"Poora design Open/Closed hai — naya storage provider ya transform ek nayi class hai jo interface implement karti hai, `ProfilePictureService` untouched. Spring ka `List<Interface>` injection iska backbone hai. Yeh `if/else` ladder se kahin behtar hai — zero regression, no merge conflicts."*
- *"Purani image cleanup: pehle DB commit, phir background job se delete — reverse order mein commit fail ho to broken image forever ban jata hai."*

---

## ⚠️ Red Flags — Don't Say These In Interview

- ❌ "Content-Type header check kar lunga" → spoofable → RCE
- ❌ "VARBINARY mein DB pe store kar dunga" → buffer-pool pollution + backup bloat + no CDN
- ❌ "Local uploads folder mein rakh dunga" → multi-server broken + crash data loss
- ❌ "Resize request thread mein kar lunga" → thread starvation under load
- ❌ "EXIF? woh kya zaroori hai" → privacy (GPS) + security leak
- ❌ "Purani image turant delete kar dunga" → commit-fail = broken image forever
- ❌ "Har provider ke liye if/else laga dunga" → OCP violation, regression + merge hell
- ❌ "Same URL rakhunga" (no version) → stale cache, DP update na dikhe

---

## 🧠 Memory Hooks Recap

- **SNAP** — Signature/Size, No-local-disk, Async-transform, Publish-via-CDN
- **"EXTEND HAAN, MODIFY NAA"** + **"USB-C PORT"** — Open/Closed Principle
- **"Magic bytes"** — file ki asli pehchaan, rename-proof
- **"Photo studio + godown + tag number"** — object storage + DB pointer
- **"Alag kamre mein editing"** — async processing

---

## 🔗 Cross-References

- For **OOP deep**: see [cheatsheets/oop-principles.md](../cheatsheets/oop-principles.md)
- For **Java↔.NET**: see [cheatsheets/spring-vs-dotnet.md](../cheatsheets/spring-vs-dotnet.md)
- For **SQL deep**: see [cheatsheets/sql-locking-isolation.md](../cheatsheets/sql-locking-isolation.md)
- For **all memory hooks**: see [cheatsheets/memory-hooks.md](../cheatsheets/memory-hooks.md)

**Self-test**: Take the [Day 12 quiz](../quizzes/day-012-upload-profile-picture.md) — 50 MCQs to verify mastery.
