# 📝 Day 12 Quiz — Upload Profile Picture (50 MCQs)

**📖 Lesson**: [2026-05-24-day12-upload-profile-picture.md](../lessons/01-beginner/2026-05-24-day12-upload-profile-picture.md)
**🎯 Topic**: Profile picture upload (validation, storage, async, CDN) + OOP Open/Closed Principle

**How to use**:
1. Replace `___` with your letter for each question (`**Your Answer**: B`).
2. Complete ALL 50 before revealing the answer key at the bottom.
3. When done, tell mentor: *"Day 12 quiz evaluate karo"*.

**Sections**: A. Code Output (10) · B. Bug Spotting (8) · C. Dry-Run (10) · D. Scenarios (10) · E. Concept (12)
**Stars**: ⭐ easy · ⭐⭐ medium · ⭐⭐⭐ hard

---

## Section A — Code Output Prediction (10)

### Q1. ⭐⭐ Yeh code chala — output kya?
```java
byte[] sig = {(byte)0xFF, (byte)0xD8, (byte)0xFF};
System.out.println(
    (sig[0] & 0xFF) == 0xFF && (sig[1] & 0xFF) == 0xD8 ? "JPEG" : "OTHER");
```
A) OTHER
B) JPEG
C) Compile error (byte cast)
D) Runtime exception

**Your Answer**: ___

---

### Q2. ⭐⭐ Multipart limit config ke saath 4MB file aaye:
```properties
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=6MB
```
Ek 4MB image POST hui (single part). Result?
A) Rejected — max-file-size exceeded
B) Accepted — within both limits
C) Rejected — max-request-size exceeded
D) Truncated to 5MB

**Your Answer**: ___

---

### Q3. ⭐⭐⭐ Angular upload progress:
```typescript
map(event => {
  if (event.type === HttpEventType.UploadProgress)
    return event.total ? Math.round(100 * event.loaded / event.total) : 0;
  return -1;
})
```
File 1,000,000 bytes; ek progress event aaya `loaded=250000, total=1000000`. Yeh emission?
A) -1
B) 250000
C) 25
D) 0

**Your Answer**: ___

---

### Q4. ⭐⭐ S3 key banayi:
```java
int version = 4;
long userId = 123;
String key = "avatars/user-%d/v%d-orig".formatted(userId, version);
System.out.println(key);
```
A) `avatars/user-123/v4-orig`
B) `avatars/user-%d/v%d-orig`
C) `avatars/user-4/v123-orig`
D) Compile error

**Your Answer**: ___

---

### Q5. ⭐⭐⭐ EXIF strip via re-encode:
```java
BufferedImage in = ImageIO.read(originalWithGps);   // EXIF + GPS present
BufferedImage out = new BufferedImage(
    in.getWidth(), in.getHeight(), BufferedImage.TYPE_INT_RGB);
out.createGraphics().drawImage(in, 0, 0, null);
// out ko PNG likho
```
`out` mein original ke EXIF GPS tags?
A) Present — copy ho jate hain
B) Absent — naya BufferedImage sirf pixels rakhta hai, metadata nahi
C) Present but encrypted
D) Depends on JVM version

**Your Answer**: ___

---

### Q6. ⭐⭐ Cache-busting URL:
```typescript
const url = '/avatars/user-123.webp';
const busted = `${url}?v=${4}`;
console.log(busted);
```
A) `/avatars/user-123.webp`
B) `/avatars/user-123.webp?v=4`
C) `/avatars/user-123.webp&v=4`
D) `/avatars/user-123.webp?version=4`

**Your Answer**: ___

---

### Q7. ⭐⭐ DI of all implementations:
```java
public ProfilePictureService(List<ImageTransform> transforms) {
    System.out.println(transforms.size());
}
```
Agar `ResizeTransform`, `WatermarkTransform`, `WebpTransform` teeno `@Component` hain, output?
A) 0
B) 1
C) 3
D) NullPointerException

**Your Answer**: ___

---

### Q8. ⭐⭐⭐ SQL OUTPUT clause:
```sql
UPDATE users
SET avatar_version = avatar_version + 1
OUTPUT inserted.avatar_version
WHERE id = 7;   -- pehle version 4 tha
```
Returned value?
A) 4
B) 5
C) 0 (no rows)
D) NULL

**Your Answer**: ___

---

### Q9. ⭐⭐ `URL.createObjectURL`:
```typescript
const a = URL.createObjectURL(file);
const b = URL.createObjectURL(file);
console.log(a === b);
```
A) true — same file = same URL
B) false — har call naya unique blob URL banata hai
C) Throws
D) undefined

**Your Answer**: ___

---

### Q10. ⭐⭐⭐ Async return type:
```java
@Async
public void process(Long id) { /* heavy 2s work */ }
// caller:
long t0 = System.currentTimeMillis();
processor.process(99L);
System.out.println(System.currentTimeMillis() - t0 < 100);
```
(@Async properly enabled, separate pool.) Output approx?
A) false — caller 2s block hota hai
B) true — call turant return, kaam background mein
C) Compile error (void @Async)
D) Always exactly 2000

**Your Answer**: ___

---

## Section B — Bug Spotting (8)

### Q11. ⭐⭐ Is upload validation mein kya galat hai?
```java
public void validate(MultipartFile file) {
    if (!"image/jpeg".equals(file.getContentType()))
        throw new InvalidUploadException("only jpeg");
}
```
A) `equals` ka order galat
B) Client-controlled `getContentType()` pe bharosa — spoofable; magic bytes check chahiye
C) Exception type galat
D) Code bilkul theek hai

**Your Answer**: ___

---

### Q12. ⭐⭐⭐ Memory/perf bug spot karo:
```java
public String upload(Long userId, MultipartFile file) {
    byte[] all = file.getBytes();                 // line X
    s3.putObject(bucket, key, new ByteArrayInputStream(all), meta);
    return key;
}
```
A) `key` undefined
B) `file.getBytes()` poori file RAM mein load karta hai — large files / concurrency pe heap spike; streaming use karo
C) `ByteArrayInputStream` invalid
D) putObject signature galat

**Your Answer**: ___

---

### Q13. ⭐⭐⭐ Cleanup order bug:
```java
@Transactional
public void replaceAvatar(Long id, String newKey) {
    String old = userRepo.getAvatarKey(id);
    storage.delete(old);              // line 1
    userRepo.updateAvatar(id, newKey); // line 2
}
```
A) `@Transactional` galat hai
B) Pehle storage delete, phir DB update — agar DB update/commit fail hua to old file gayi aur naya bhi nahi → broken image. Commit-then-delete karo.
C) `getAvatarKey` slow hai
D) Koi bug nahi

**Your Answer**: ___

---

### Q14. ⭐⭐ Angular leak spot karo:
```typescript
onSelect(file: File) {
  this.previewUrl = URL.createObjectURL(file);  // har select pe
}
// ngOnDestroy missing
```
A) `createObjectURL` deprecated
B) Purani blob URL kabhi `revokeObjectURL` nahi hoti → memory leak (browser holds blobs)
C) `previewUrl` type galat
D) Theek hai

**Your Answer**: ___

---

### Q15. ⭐⭐⭐ DoS bug spot karo:
```java
BufferedImage img = ImageIO.read(file.getInputStream()); // koi guard nahi
int w = img.getWidth(), h = img.getHeight();
```
A) `ImageIO.read` returns null kabhi kabhi — woh main bug
B) Dimension/size guard nahi — malicious `60000x60000` header se ~14GB alloc → OOM (decompression bomb). Decode se pehle guard chahiye.
C) `getWidth` slow
D) Theek hai

**Your Answer**: ___

---

### Q16. ⭐⭐ OCP bug spot karo:
```java
public String store(String provider, byte[] data, String key) {
    if (provider.equals("s3")) { /*...*/ }
    else if (provider.equals("azure")) { /*...*/ }
    return key;
}
```
A) `byte[]` parameter galat naam
B) Provider `if/else` ladder — har naya provider is tested method ko modify karwata hai (OCP violation). Interface + polymorphism use karo.
C) `return key` galat jagah
D) Theek hai

**Your Answer**: ___

---

### Q17. ⭐⭐ Ownership bug:
```java
@PutMapping("/avatar/{userId}")
public void update(@PathVariable Long userId, @RequestParam MultipartFile f) {
    service.upload(userId, f);   // userId path se
}
```
A) `@PutMapping` galat
B) `userId` path se aata hai — main kisi aur ka userId de ke uski DP badal sakta hun (IDOR). Logged-in user (`@AuthenticationPrincipal`) se lo.
C) `MultipartFile` galat
D) Theek hai

**Your Answer**: ___

---

### Q18. ⭐⭐⭐ Spring self-invocation bug:
```java
@Service
public class Svc {
    public void upload(...) { this.process(...); }  // same-class call
    @Async public void process(...) { /* heavy */ }
}
```
A) `@Async` typo
B) Same-class call proxy ko bypass karta hai → `process` synchronously chalega (no async). Alag bean se call karo ya event publish karo.
C) `process` private hona chahiye
D) Theek hai

**Your Answer**: ___

---

## Section C — Dry-Run Tracing (10)

### Q19. ⭐⭐⭐ Trace karo:
```
Initial: users row id=5, avatar_version=2, avatar_key='v2'
T+0:  Request A: UPDATE ... version=version+1, key='v3a' WHERE id=5  (X-lock)
T+1:  Request B: UPDATE ... version=version+1, key='v3b' WHERE id=5  (waits for lock)
T+2:  A commits
T+3:  B proceeds
Question: Final avatar_version aur key?
```
A) version=3, key='v3a'
B) version=3, key='v3b'
C) version=4, key='v3b'
D) version=4, key='v3a'

**Your Answer**: ___

---

### Q20. ⭐⭐ Trace pipeline:
```
transforms = [ResizeTransform, WatermarkTransform]  (@Order 1, 2)
img0 = 4000x3000
ResizeTransform(maxDim=1080): scale to 1080x810
WatermarkTransform: adds logo (same dims)
Question: final dims?
```
A) 4000x3000
B) 1080x810
C) 1080x1080
D) 810x1080

**Your Answer**: ___

---

### Q21. ⭐⭐⭐ Trace presigned flow:
```
T+0:  Browser → backend: "presign for key avatars/u9/v1"
T+5:  backend signs URL (valid 300s), returns
T+10: Browser → S3: PUT file using signed URL
T+12: S3 stores, emits ObjectCreated
T+15: backend worker reads event → resize
Question: Actual file bytes kis path se gaye?
```
A) Browser → backend → S3
B) Browser → S3 directly (signed URL pe), backend ko bytes nahi mile
C) Backend → S3 → Browser
D) S3 → backend → Browser

**Your Answer**: ___

---

### Q22. ⭐⭐⭐ Trace validation chain:
```
validators = [SizeValidator, ContentTypeValidator, DimensionValidator]
Input: 3MB file, real type = application/zip (renamed .jpg), dims n/a
SizeValidator: 3MB < 5MB → pass
ContentTypeValidator: tika.detect → "application/zip" not in allowed → THROW
Question: DimensionValidator chalega?
```
A) Haan, sab chalte hain
B) Nahi — ContentTypeValidator throw karta hai, chain wahin ruk jati hai
C) Sirf DimensionValidator chalega
D) Sab skip ho jate hain

**Your Answer**: ___

---

### Q23. ⭐⭐ Trace cache-busting:
```
T+0:  DP url = /av/u1.webp?v=3  (CDN cached v3 image)
T+1:  user uploads new, DB version → 4
T+2:  UI sets url = /av/u1.webp?v=4
Question: Browser ko konsi image milegi at T+2?
```
A) Cached v3 (same base path)
B) Fresh fetch — `?v=4` naya URL hai, CDN/browser cache miss → new image
C) Error
D) Default placeholder

**Your Answer**: ___

---

### Q24. ⭐⭐⭐ Trace async worker retry (at-least-once queue):
```
T+0:  worker picks job(userId=2, key='v5')
T+1:  resize ok, write thumb, but crash BEFORE ack
T+2:  queue redelivers same job
T+3:  worker picks job again, resize, write thumb 'v5' (overwrite)
Question: Outcome agar worker idempotent hai?
```
A) Duplicate thumbnails, corruption
B) Same 'v5' thumb overwrite — no duplicate, correct final state (idempotent)
C) Infinite loop
D) Data loss

**Your Answer**: ___

---

### Q25. ⭐⭐ Trace memory:
```
onSelect call 1: previewUrl = blob:#a  (no revoke)
onSelect call 2: revoke(previewUrl=blob:#a); previewUrl = blob:#b
ngOnDestroy:     revoke(previewUrl=blob:#b)
Question: Kitne blob URLs leak (un-revoked) hue?
```
A) 2
B) 1
C) 0
D) 3

**Your Answer**: ___

---

### Q26. ⭐⭐⭐ Trace orphan cleanup:
```
T+0:  image_asset: (key='v1', ACTIVE)
T+1:  new upload txn: UPDATE v1→ORPHAN; INSERT v2 ACTIVE; COMMIT
T+2:  cleanup job: scan status='ORPHAN' AND age>1day → none yet (v1 just now)
T+3:  next day: cleanup finds v1 ORPHAN age>1day → storage.delete('v1'); mark DELETED
Question: T+1 ke baad active image_asset rows?
```
A) v1 ACTIVE only
B) v2 ACTIVE only (v1 is ORPHAN)
C) Both ACTIVE
D) None

**Your Answer**: ___

---

### Q27. ⭐⭐ Trace 202 response timing:
```
T+0:    POST avatar
T+20ms: validate + store original + DB pointer set
T+21ms: publish AvatarUploadedEvent (async)
T+22ms: return 202 ACCEPTED "PROCESSING"
T+650ms: worker finishes resize/thumbnails
Question: User ko HTTP response kab milta hai?
```
A) T+650ms (resize ke baad)
B) ~T+22ms (resize ka wait nahi)
C) T+0
D) Never (async)

**Your Answer**: ___

---

### Q28. ⭐⭐⭐ Trace OCP extension:
```
Before: StorageProvider impls = [S3StorageProvider @Primary]
Task: add Azure. Steps taken:
  1. new class AzureBlobStorageProvider implements StorageProvider
  2. move @Primary to Azure
Question: ProfilePictureService.java mein kitni lines change hui?
```
A) Several (provider switch)
B) Zero — service interface se baat karti hai, impl swap config-level
C) Entire method rewritten
D) Compile error

**Your Answer**: ___

---

### Q29. ⭐⭐ Trace buffer-pool reasoning:
```
DB stores avatars as VARBINARY(MAX). 1M users × 2MB each.
Buffer pool (RAM cache) = 16GB.
Question: Likely effect on query latency for hot user rows?
```
A) Faster — sab cached
B) Slower — large LOBs cache mein jagah lete hain, hot rows/indexes evict, more disk reads
C) No effect
D) Queries fail

**Your Answer**: ___

---

### Q30. ⭐⭐⭐ Trace dimension guard:
```
File header claims 50000 x 50000 (PNG). Guard: maxPixels = 40_000_000.
Check: 50000 * 50000 = 2,500,000,000 > 40,000,000 → ?
```
A) Decode proceeds, OOM risk
B) Reject before full decode (exceeds maxPixels) — DoS prevented
C) Resize to 40M pixels
D) Crash immediately

**Your Answer**: ___

---

## Section D — Scenario-Based Decisions (10)

### Q31. ⭐⭐ Production issue: FoodPanda pe 3 app servers behind LB. Users report "DP kabhi dikhti, kabhi nahi". Files `/var/app/uploads/` mein save hote hain. Sabse likely cause?
A) Slow network
B) Local disk per-server — upload ek server pe, read doosre se → file not found. Object storage use karo.
C) Browser cache
D) CDN down

**Your Answer**: ___

---

### Q32. ⭐⭐⭐ Security review: ek `.jpg` upload ke baad attacker `site.com/uploads/x.jpg?cmd=...` se commands chala raha. Root cause + fix?
A) Weak password; reset it
B) File real type validate nahi (magic bytes), web-executable path pe original naam se save → uploaded shell executed. Magic-byte validate + non-executable storage (object store) + random keys.
C) SQL injection
D) CSRF

**Your Answer**: ___

---

### Q33. ⭐⭐ Privacy complaint: user ne ghar se DP upload ki, kisi ne uska address nikal liya image se. Cause?
A) DB leak
B) EXIF metadata (GPS) strip nahi hui upload pe. Re-encode/strip karo.
C) Weak TLS
D) Logging

**Your Answer**: ___

---

### Q34. ⭐⭐⭐ Scale decision: Instagram-scale uploads, app servers bandwidth pe choke. Best move?
A) Bigger app servers forever
B) Presigned URL direct-to-S3 upload — browser seedha S3, backend sirf sign kare; processing S3 event se async
C) Store in DB
D) Disable uploads

**Your Answer**: ___

---

### Q35. ⭐⭐ Decision: resize 12MB photos request thread pe ho raha, peak pe API slow. Best fix?
A) Remove resize
B) Async pipeline (queue + worker), return 202; thread free rahe
C) Bigger thread pool only
D) Resize on client only (trust client)

**Your Answer**: ___

---

### Q36. ⭐⭐⭐ Decision: naya requirement — AVIF format support add karna hai existing image pipeline mein. Tested code minimal touch chahiye. Approach?
A) Pipeline method mein `if (avif)` branch add karo
B) Naya `AvifTransform implements ImageTransform` class banao, pipeline list mein register — existing untouched (OCP)
C) Saari transforms rewrite
D) Naya service banao duplicate

**Your Answer**: ___

---

### Q37. ⭐⭐ Decision: nayi DP upload hui par users ko purani dikh rahi. Sabse seedha fix?
A) DB index add
B) Cache-busting — URL mein version/hash add karo (`?v=N`)
C) CDN band karo
D) Force refresh user se kehna

**Your Answer**: ___

---

### Q38. ⭐⭐⭐ Decision: avatar replace pe storage cost barh rahi (purani files orphan). Safe cleanup strategy?
A) Upload ke time purani turant delete (sync)
B) DB commit first; mark old ORPHAN; scheduled idempotent job deletes ORPHAN files after grace period
C) Kabhi delete mat karo
D) Manual deletion monthly

**Your Answer**: ___

---

### Q39. ⭐⭐ Decision: ek hi storage provider hai (S3), team interface bana rahi har cheez ke liye "future-proofing". Senior call?
A) Haan, har cheez abstract karo
B) Abhi concrete theek hai (YAGNI) — interface tab nikaalo jab doosri real implementation aaye (Rule of Three)
C) Microservice bana do
D) Reflection use karo

**Your Answer**: ___

---

### Q40. ⭐⭐⭐ Decision: 100 concurrent uploads, har 5MB, server heap 2GB. Code `file.getBytes()` use karta hai. Risk + fix?
A) No risk
B) ~500MB+ instant heap (100×5MB buffered) → GC pressure/OOM. Stream via `getInputStream()` to storage; don't buffer whole file.
C) CPU bound only
D) Disk only

**Your Answer**: ___

---

## Section E — Concept Mastery (12)

### Q41. ⭐ Magic bytes kyun, extension/Content-Type kyun nahi?
A) Extension lamba hai
B) Extension aur Content-Type client-controlled (spoofable); magic bytes file ke andar ki actual signature hain
C) Magic bytes fast hain
D) Koi farq nahi

**Your Answer**: ___

---

### Q42. ⭐⭐ Object storage vs DB BLOB — DB mein binary kyun bura?
A) DB binary support nahi karta
B) Buffer-pool pollution, backup bloat, replication lag, no CDN — DB mein pointer rakho, binary object store mein
C) Object store sasta nahi
D) BLOB illegal hai

**Your Answer**: ___

---

### Q43. ⭐ Open/Closed Principle ka one-liner?
A) Sab classes public
B) Open for extension, closed for modification — naya behavior naye code se, tested code edit kiye bina
C) Always inherit
D) Never use interfaces

**Your Answer**: ___

---

### Q44. ⭐⭐ OCP achieve karne ka primary mechanism?
A) Static methods
B) Polymorphism — varying behavior ko interface ke peeche, high-level interface se depend kare (Strategy + DI)
C) Global variables
D) Reflection only

**Your Answer**: ___

---

### Q45. ⭐⭐ EXIF strip ka side benefit (privacy ke alawa)?
A) File chhoti dikhti
B) Re-encode embedded payloads/malicious metadata bhi remove karta hai (security)
C) Color badalta
D) Koi nahi

**Your Answer**: ___

---

### Q46. ⭐⭐ Presigned URL kya hai?
A) Permanent public URL
B) Temporary signed URL jisse browser seedha storage pe limited operation (PUT) kar sake; backend sirf sign karta hai
C) DB connection string
D) CDN cache key

**Your Answer**: ___

---

### Q47. ⭐⭐⭐ `javax.imageio.ImageIO` OCP example kaise hai?
A) Singleton hai
B) Naye format readers `ImageReaderSpi` plugins se `ServiceLoader` ke through add hote hain — ImageIO ka source touch kiye bina (extend, not modify)
C) Sirf JPEG karta hai
D) Final class hai

**Your Answer**: ___

---

### Q48. ⭐⭐ Spring `List<Interface>` injection OCP mein kya role?
A) Koi nahi
B) Saari implementations auto-inject hoti hain → nayi impl (`@Component`) add karna existing wiring/service modify nahi karta
C) Performance ke liye
D) Sirf testing

**Your Answer**: ___

---

### Q49. ⭐⭐ OCP kab jaan-boojh ke chhodte ho?
A) Hamesha follow
B) Jab sirf ek implementation hai aur doosri ki clear zaroorat nahi — interface = premature abstraction (YAGNI). Concrete first.
C) Production mein kabhi nahi
D) DTOs mein hamesha

**Your Answer**: ___

---

### Q50. ⭐⭐⭐ Angular `reportProgress: true` + `observe: 'events'` kya enable karta hai?
A) Faster upload
B) Raw `HttpEvent` stream including `HttpUploadProgress` (loaded/total) — progress bar ke liye, sirf final response ke bajaye
C) Auto-retry
D) Caching

**Your Answer**: ___

---

<details>
<summary>⚠️ Click to reveal answer key — complete all 50 first!</summary>

### Section A: Code Output Prediction
1. **B (JPEG)** — `0xFF,0xD8,0xFF` is the JPEG SOI magic signature; masking with `& 0xFF` makes the byte comparison correct → prints JPEG.
2. **B** — 4MB ≤ 5MB (file) and ≤ 6MB (request), so accepted.
3. **C (25)** — `100 * 250000 / 1000000 = 25`, rounded → 25.
4. **A** — `formatted` substitutes `%d` in order: userId=123, version=4.
5. **B** — A new `BufferedImage` holds only pixel data; drawing copies pixels, not metadata → EXIF/GPS gone.
6. **B** — Template literal appends `?v=4` to the base URL.
7. **C (3)** — Spring injects all three `@Component` `ImageTransform` beans into the `List`.
8. **B (5)** — `OUTPUT inserted.*` returns the post-update value (4→5).
9. **B (false)** — Each `createObjectURL` call returns a new unique blob URL (must revoke each).
10. **B (true)** — `@Async` returns immediately; heavy work runs on the pool, so elapsed < 100ms.

### Section B: Bug Spotting
11. **B** — `getContentType()` is client-supplied and spoofable; use magic-byte detection.
12. **B** — `file.getBytes()` buffers the entire file in heap; stream via `getInputStream()` instead.
13. **B** — Deletes old file before DB commit; if commit fails, old is gone and new isn't recorded → broken image. Commit first, then delete (async).
14. **B** — No `revokeObjectURL`; each preview leaks a blob URL (browser retains the blob).
15. **B** — No dimension/size guard before decode; a huge declared size triggers giant allocation → OOM (decompression bomb).
16. **B** — `if/else` provider ladder violates OCP; new providers force edits to tested code. Use interface + polymorphism.
17. **B** — `userId` from the path enables IDOR (change anyone's avatar). Use the authenticated principal.
18. **B** — Same-class call bypasses the Spring proxy, so `@Async` runs synchronously. Call via another bean or publish an event.

### Section C: Dry-Run Tracing
19. **C (version=4, key='v3b')** — A: 2→3 key v3a (commit). B then runs on committed state: 3→4 key v3b. (Lost-update on key is why version+atomicity matter; final reflects B last.)
20. **B (1080x810)** — Resize scales longest side to 1080 (4000→1080, 3000→810); watermark keeps dims.
21. **B** — Presigned upload sends bytes browser→S3 directly; backend never receives the file bytes.
22. **B** — Validators run in order; `ContentTypeValidator` throws, halting the chain before `DimensionValidator`.
23. **B** — `?v=4` is a new URL → cache miss → fresh image fetched.
24. **B** — Idempotent worker overwrites the same `v5` key; redelivery causes no duplication, final state correct.
25. **C (0)** — Call 2 revokes #a, destroy revokes #b; none leaked.
26. **B** — After the txn, v1 is ORPHAN and v2 is ACTIVE → only v2 ACTIVE.
27. **B (~T+22ms)** — Response returns at 202 (after store + pointer + publish); resize happens after, not awaited.
28. **B (Zero)** — Service depends on the `StorageProvider` interface; adding Azure is a new class + `@Primary` swap, no service edits.
29. **B** — Large LOBs occupy buffer-pool pages, evicting hot rows/indexes → more disk reads, slower queries.
30. **B** — `50000*50000` exceeds maxPixels → reject before full decode; DoS prevented.

### Section D: Scenario-Based Decisions
31. **B** — Local disk is per-server; uploads land on one node, reads hit another → intermittent missing files. Use object storage.
32. **B** — No real-type validation + executable web path with original name → uploaded shell executed. Validate magic bytes, store in non-executable object storage with random keys.
33. **B** — EXIF GPS wasn't stripped; re-encode/strip on upload.
34. **B** — Presigned direct-to-S3 upload offloads app-server bandwidth; process via S3 events asynchronously.
35. **B** — Move resize to an async queue+worker, return 202; keeps request threads free.
36. **B** — New `AvifTransform implements ImageTransform`, register in the pipeline; existing code untouched (OCP).
37. **B** — Add cache-busting version/hash to the URL.
38. **B** — Commit DB first, mark old ORPHAN, scheduled idempotent job deletes after a grace period.
39. **B** — One implementation doesn't need an interface (YAGNI); abstract on the second real implementation (Rule of Three).
40. **B** — `getBytes()` buffers 100×5MB ≈ 500MB+ → GC pressure/OOM; stream to storage instead.

### Section E: Concept Mastery
41. **B** — Extension and Content-Type are client-controlled/spoofable; magic bytes are the file's actual signature.
42. **B** — DB BLOBs cause buffer-pool pollution, backup bloat, replication lag, and no CDN; store a pointer, keep binary in object storage.
43. **B** — Open for extension, closed for modification.
44. **B** — Polymorphism: hide what varies behind an interface, depend on it (Strategy + DI).
45. **B** — Re-encoding also strips embedded malicious payloads/metadata (security bonus).
46. **B** — A temporary signed URL letting the browser perform a limited operation (e.g., PUT) directly on storage; backend only signs.
47. **B** — New format readers plug in via `ImageReaderSpi`/`ServiceLoader` without modifying ImageIO source (extend, not modify).
48. **B** — All implementations auto-inject; adding a new `@Component` impl doesn't modify existing wiring/service.
49. **B** — When there's a single implementation with no clear second use, an interface is premature (YAGNI); write concrete first.
50. **B** — It yields the raw `HttpEvent` stream including `HttpUploadProgress` (loaded/total) for a progress bar, not just the final response.

</details>

---

## 📊 Scoring Guide
- **45-50**: 🏆 Senior-ready — upload security + storage + OCP solid.
- **38-44**: ✅ Strong — revise magic-bytes/orphan-order edge cases.
- **30-37**: ⚠️ Re-read §Stack-1 (validation/streaming) + OOP (OCP).
- **<30**: 🔴 Re-study the full lesson, then retake.
