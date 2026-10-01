# AGENTS.md

## Proje ve stack

TypeScript · Next.js · pnpm · Vitest · veritabanı: PostgreSQL + Drizzle

Bu kurallar bu repoda çalışan her kodlama ajanı için geçerlidir. Bir kural mevcut kodla çelişiyorsa kodu değiştirmeden önce sor.

- Next.js (App Router) ve TypeScript. Kod `src/` altındadır, sayfalar `src/app` içindedir.
- Component varsayılan olarak Server Component'tir. `'use client'` sadece etkileşim gereken yaprak component'e yazılır.
- Framework'ün zorunlu kıldığı dosya adları (`page.tsx`, `layout.tsx`, `route.ts`) kebab-case kuralının dışındadır. Diğer `.tsx` dosyaları da kebab-case yazılır (`user-card.tsx`).

## Komutlar

- Kurulum: `pnpm install`
- Geliştirme sunucusu: `pnpm dev`
- Test: `pnpm exec vitest run`
- Tip kontrolü: `pnpm exec tsc --noEmit`
- Lint: `pnpm exec eslint .`

- Build: `pnpm build`

## Proje yapısı

```
src/
├── app/          sayfalar, route handler'lar, server action'lar
├── components/   ekranda tekrar kullanılan parçalar
├── services/     iş mantığı, framework import etmez
├── db/           veritabanı erişimi ve sorgular
└── env.ts        ortam değişkenleri, açılışta doğrulanır
```

Yukarıdaki ağaç beklenen düzendir. Repodaki klasörler farklıysa repodaki düzeni izle ve bana söyle. Yapıya uymayan yeni bir klasör gerekiyorsa önce sor.

## Yeni iş eklerken

- Şema değişikliği: önce şemayı değiştir, sonra Veri modelleme bölümünde yazan yolla migration ya da dönüşüm dosyasını üret. Uygulanmış olana dokunma.

- Yeni uç nokta: giriş `src/app` (route handler ya da server action), iş mantığı `src/services` içinde. Girdiyi sınırda doğrula, yetkiyi sunucuda kontrol et, en az bir başarılı ve bir hatalı durum testi yaz.

- Yeni ekran: önce veriyi hangi component'in yükleyeceğine ve etkileşimin sahibine karar ver, sonra yükleniyor, boş ve hata hallerini yaz.

- Yeni ortam değişkeni: önce Ortam değişkenleri bölümündeki okuma yerine, sonra `.env.example` dosyasına aynı değişiklikte ekle.
- Yeni klasör ya da komut eklersen bu dosyanın ilgili bölümüne tek satır eklemeyi diff olarak göster, onayımla yaz.

## Kod kuralları

- Değişken ve fonksiyon adları camelCase (`getUserById`), tip ve sınıf adları PascalCase (`User`, `CreateUserInput`), sabitler UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`).
- Dosya ve klasör adları kebab-case (`user-service.ts`). Framework'ün dayattığı adlar (`index.ts` gibi) bu kuralın dışındadır.
- Boolean adı `is`, `has` ya da `can` ile başlar (`isActive`). Fonksiyon adı fiil (`createUser`), tip adı isim (`User`) olur.
- Tip adına `I` öneki ya da `Type` soneki ekleme (`IUser`, `UserType`). Önek ve sonek adı uzatır, bilgi eklemez.

- Proje içi import'larda `@/` alias'ı kullan: `import { createUser } from '@/services/user-service'`. Alias `tsconfig.json` içinde tanımlıdır, Next.js ek ayar istemeden çözer:
  ```json
  { "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
  ```
- `../` ile relative import kullanma. Dosya taşındığında yol kırılır. Aynı klasördeki dosya için `./` serbest.

- Akış tek yönlüdür: `src/app` (sayfa, route handler, server action) → `src/services` → `src/db`. Üst katman alttakini çağırır, tersi olmaz.
- Component ve route handler veritabanını doğrudan çağırmaz, sadece service çağırır. Sorgu `src/db` içinde yaşar, çünkü aynı sorguyu birden fazla yerde kopyalamak tutarsızlık üretir.
- `src/services` içindeki iş mantığı `next/*` ve `react` import etmez. Cookies, redirect, revalidate gibi framework koduna `src/app` içinde yaz, çünkü service framework'süz test edilebilmeli.

- Mantığı saf fonksiyonlara koy: aynı girdi aynı çıktıyı verir, global durum okumaz. DB, ağ, dosya ve saat erişimi katmanın kenarında kalır.
- Bağımlılığı import edilmiş singleton yerine parametre olarak al, böylece test sahte bağımlılığı doğrudan geçirir:
  ```ts
  export async function createUser(users: UserStore, input: CreateUserInput): Promise<User> { /* ... */ }
  ```
- Testin kurulumu 10 satırı aşıyorsa fonksiyonu böl, çünkü fazla iş yapıyordur.

- Bir component veriyi ya kendisi yükler ya da prop olarak alır, ikisini birden yapmaz.
- Olay handler'ı, olayın sonucunu bilen component'te yazılır. Bir callback'i iki katmandan fazla prop olarak taşıyorsan component sınırını yeniden çiz.
- Başka state'ten ya da prop'tan hesaplanabilen değeri state'e koyup `useEffect` ile eşitleme, render sırasında hesapla. `useEffect` sadece dış sistemle senkron içindir.
- Sekme, filtre, sayfa ve açık detay paneli gibi paylaşılabilir durum URL'de tutulur, böylece link ve geri tuşu çalışır.
- Veri gösteren her bölümün yükleniyor, boş ve hata hali vardır. Birini atlarsan kullanıcı boş ekran görür.
- Alt parçaları ayarlamak için `cardTitle`, `cardActions` gibi prop biriktirme. `children` ya da alt component'lerle (`<Card.Header>`) kur.
- Arayüz yavaşsa tahminle düzeltme. `react-perf-audit` skill'i re-render, istek, bundle ve Web Vitals sorunlarını `dosya:satır` ile raporlar; önce onu çalıştır, düzeltmeyi rapora göre yap.

- Veriyi Server Component'te yükle, etkileşim gereken yaprak Client Component'e prop olarak geçir. Birbirine bağlı olmayan yüklemeleri `Promise.all` ile aynı anda başlat, art arda `await` etme. Client Component içinde veri yüklemek için `useEffect` yazma.
- Route segmentine `loading.tsx` ve `error.tsx` ekle (`error.tsx` `'use client'` ister). Bağımsız yüklenen bölümü ayrıca kendi `<Suspense>` sınırıyla sar.
- Ağır istemci kütüphanesini (grafik, zengin editör) `next/dynamic` ile yükle, `'use client'` sınırını küçük tut. İstemciye giden her import sayfanın yükleme süresine eklenir.

- Bu bölümdeki kurallar yazdığın ya da değiştirdiğin kod içindir. Dokunmadığın mevcut kodu bu kurallara uydurmak için yeniden düzenleme.
- Ortak soyutlamayı (helper, base class, ortak modül) aynı kalıp üçüncü kez tekrar edince çıkar. Erken çıkan soyutlama ilk iki kullanıma göre biçimlenir, üçüncüsü uymayınca herkes onu eğip bükmek zorunda kalır.
- Mevcut bir soyutlamaya tek bir kullanım için `if`, bayrak parametresi ya da özel seçenek eklemen gerekiyorsa ekleme; bunu bana söyle ve soyutlamayı bölmeyi öner. Cevap gelmeden ilerleyemiyorsan dur ve sor.
- Yeni iş mevcut yapıya zorla sığıyorsa önce yapıyı değişikliğe uygun hale getiren, davranışı değiştirmeyen ayrı bir adım öner, sonra değişikliği yap. Bu adımı onayım olmadan yapma.

## Veri modelleme

- Şema `src/db/schema.ts` içindedir, tek kaynak odur. Değişiklikten sonra `pnpm exec drizzle-kit generate` ile migration üret, `pnpm exec drizzle-kit migrate` ile uygula. Özel SQL gerekiyorsa `pnpm exec drizzle-kit generate --custom` ile boş migration üretip SQL'i oraya yaz.
- Tablo adı snake_case çoğul (`users`), kolon adı snake_case (`created_at`). `drizzle.config.ts` ve `drizzle()` çağrısında `casing: 'snake_case'` ayarlıdır, kodda alanlar camelCase (`createdAt`) yazılır.
- Satır tiplerini şemadan türet: `type User = typeof users.$inferSelect`, `type NewUser = typeof users.$inferInsert`.
- Kolon ya da tablo silmeden önce bana sor. Geri almak için ters değişikliği yeni migration olarak üret.

- Sistemin dışından gelen girdiyi (request body, ortam değişkeni) zod ile doğrula. zod projede yoksa eklemeden önce bana sor. İç fonksiyonlar doğrulanmış veriye güvenir, tekrar doğrulamaz.
- Girdi tipini şemadan türet ve `CreateUserInput` gibi adlandır: `type CreateUserInput = z.infer<typeof createUserSchema>`. `User` gibi veritabanı tipleri bundan ayrı kalır.
- `safeParse` kullan, hatayı girdi hatası olarak döndür (HTTP yanıtında 400). `parse` ile yakalanmamış istisna fırlatma.

## Güvenlik ve performans

- Kimlik gerektiren her giriş noktası kimliği ve yetkiyi sunucuda kontrol eder: "bu kullanıcı bu kaydı görebilir mi, değiştirebilir mi?". Arayüzde butonu gizlemek koruma değildir. Webhook'ta imzayı doğrula.
- Hata yanıtında stack trace, SQL ya da dosya yolu gösterme: ayrıntıyı logla, kullanıcıya genel mesaj ve doğru durum kodu dön. Loglara token, parola ve kişisel veri (e-posta, telefon) yazma.

- Server Action da herkese açık bir POST uç noktasıdır. Yetki kontrolü sayfada değil, action'ın içinde yapılır.

- Kullanıcıdan gelen HTML'i `dangerouslySetInnerHTML` ile temizlemeden basma; projede DOMPurify gibi bir temizleyici yoksa eklemeden önce bana sor. Kullanıcının verdiği URL'i `new URL()` ile ayrıştır ve protokolün `http:` ya da `https:` olduğunu doğrula, `javascript:` linki kod çalıştırır.

- Kullanıcıdan gelen değeri sorguya string birleştirerek koyma. Parametreli sorgu ya da ORM'in sorgu kurucusunu kullan, çünkü birleştirilen string SQL injection açar.

- Döngü içinde sorgu atma (N+1). İlişkili kayıtları toplu al, sorgu sayısı satır sayısıyla artmasın: `db.query.users.findMany({ with: { posts: true } })` ile (ilişki `relations()` ile tanımlı ve şema `drizzle()`'a verilmiş olmalı).
- Liste döndüren her sorgu ve uç nokta sayfalıdır: varsayılan bir `limit` ve üst sınırı vardır. Sınırsız liste veri büyüyünce yavaşlar.
- Yeni bir sorgu sık filtrelenen, sıralanan ya da ilişki kurulan bir alana dayanıyorsa ve orada index yoksa, index'i aynı işte ekle. Index de bir şema değişikliğidir, projede şema neyle değişiyorsa onunla ekle.
- Bunun dışında ölçmeden optimizasyon yapma. Yavaşlık iddiasını sorgu sayısı ya da süreyle göster.

## Ortam değişkenleri

- Kullanılan her değişken `.env.example` içinde adıyla ve tek satır açıklamayla durur, değeri olmadan.

- Değişkenleri açılışta `src/env.ts` içinde zod ile doğrula. Sunucu kodu sadece bu dosyadan okur, başka dosyada `process.env` yazma; client component bu dosyayı import etmez. `NEXT_PUBLIC_` önekli değişken tarayıcıya gider, secret koyma.

## Test stratejisi

- Unit test saf iş mantığını test eder: hızlı çalışır, dış bağımlılığı yoktur.
- Integration test birden fazla katmanı gerçek bağımlılıklarıyla birlikte test eder. Veritabanı varsa gerçek bir test veritabanı kullan, çünkü sorgu hataları mock'ta görünmez.
- Sadece dış servisleri mock'la (e-posta, ödeme, üçüncü parti HTTP). Kendi kodunu mock'larsan test kodu değil mock'u doğrular.
- Bug düzeltirken önce bug'ı yakalayan testi yaz ve başarısız olduğunu gör, sonra düzelt. Böylece testin gerçekten o bug'ı yakaladığını bilirsin.

- Test dosyası kaynağın yanında durur, adı `*.test.ts`: `user-service.ts` → `user-service.test.ts`.
- Tek dosyayı `pnpm exec vitest run src/services/user-service.test.ts` ile çalıştır, her değişiklikte tüm paketi çalıştırma.
- Mock API'si: `vi.fn()`, `vi.mock()`, `vi.spyOn()`.

- `@/` alias'ını `vitest.config.ts` içinde `resolve.alias` ile tanımla: `resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } }`. Vitest `tsconfig.json` içindeki `paths` ayarını okumaz, alias'sız import çözülmez.
- Bunun için ek paket gerekmez.

- `playwright.config.ts` varsa uçtan uca testleri Playwright ile `e2e/` altında `*.e2e.ts` adıyla yaz (`testMatch: '**/*.e2e.ts'`), böylece unit test aracı bu dosyaları toplamaz. Config yoksa e2e aracı eklemeden önce bana sor.
- Çalıştırma: `pnpm exec playwright test`, tek dosya için `pnpm exec playwright test e2e/login.e2e.ts`. Sadece kritik akışları (giriş, ödeme, kayıt) kapsa.

## Kritik kurallar

- .env ve secret dosyalarını asla commit etme.
- Mevcut migration dosyalarını elle düzenleme, değişiklik için yeni migration üret.
- Yeni paket eklemeden önce bana sor.
- Yeni dosya açmadan önce mevcut kodu oku, benzer bir yapı varsa onu kullan.
- İstenmeyen refactor yapma, sadece istenen değişikliği yap.
- Testi silerek, atlayarak ya da beklentisini gevşeterek geçirme.

## Git

- Commit ve push'u ben istemedikçe yapma.
- Commit mesajı neyin neden değiştiğini söyler: `fix: indirimli toplamda yuvarlamayı düzelt`.

## Doğrulama

Bir görevi "bitti" saymadan önce bu komutları bu sırayla yeniden çalıştır. Hızlı biten kontroller başta, böylece hata erken çıkar. Hepsi geçmeden "bitti" deme. Son geçen koşudan beri hiçbir dosya değişmediyse o sonucu kullanabilirsin. prove-it skill'i bu listeyi okur.

```bash
pnpm exec tsc --noEmit
pnpm exec eslint .
pnpm exec vitest run
pnpm build
```

- Çalıştırmadığın komut için "geçti" deme. Listedeki bir araç kurulu değilse kurma; o maddeyi ⚠️ "doğrulanmadı" yaz ve bana sor.
- Bir komut başarısız olursa hatanın ilk satırını aynen alıntıla. Düzeltemiyorsan "bitti" yerine neyin kırık olduğunu yaz.
- Bitirirken bu formatta, sade dille yaz: toplam en fazla 10 satır, en fazla 6 madde, önce ❌ sonra ⚠️ sonra ✅. Süreç anlatma:
  ```
  Ne yaptım: tek satır, teknik olmayan biri de anlasın
  Kontrol listesi:
  - ❌ sonuç · `komut` → hatanın ilk satırı
  - ⚠️ sonuç · doğrulanmadı: neden · kontrol: tek adım
  - ✅ sonuç · kanıt: `komut` → çıktı ya da dosya:satır
  Değişenler: a, b ya da yok
  Senden beklenen: sadece senin yapman gereken bir şey varsa
  ```
