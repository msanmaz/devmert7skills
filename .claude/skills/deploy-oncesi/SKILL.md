---
name: deploy-oncesi
description: Deploy, yayın, merge ya da release kararı sorulduğunda devreye girer: mevcut diff'i ve AGENTS.md'yi okuyup kanıtlı bir kontrol listesi üretir (doğrulama komutları, env değişkenleri, migration, secret taraması, bağımlılıklar, rollback notu, güvenlik ve sunucu tarafı veri riskleri) ve tek satırlık bir karar verir. Deploy etmez, sadece kontrol eder. İş "bitti" denmiş olsa bile soru deploy ise bu skill kullanılır. Şu durumlarda kullan: "deploy edelim mi", "yayına hazır mı", "canlıya alalım mı", "production'a çıkmadan kontrol et", "merge etmeden önce kontrol", "release öncesi", "pre-deploy check", "canlıya almadan bir bak". Şu durumlarda kullanma: performans denetimi ("neden yavaş", "bundle size", "re-render", "PR'ı performans için incele"; bunlar react-perf-audit'in işi), deploy sürecinin genel olarak nasıl çalıştığını anlatma, sıfırdan CI kurma, zaten başarısız olmuş bir deploy'un loglarını ayıklama.
---

# deploy-oncesi

## Amaç
Canlıya giden hiçbir şey kontrolsüz gitmesin: diff'ten ve AGENTS.md'den kanıtlı bir yayın öncesi liste çıkar, karar ver, dur.

Ana kural: deploy, push, publish, production migration ve release komutlarını bu skill içinde çalıştırma. Kullanıcı istese bile komutu ona ver, o çalıştırsın.

## Durumlar
Her kontrol şu dört durumdan birini alır ve bir kanıt taşır (komut çıktısının özeti ya da `dosya:satır`):
- ✅ geçti, kanıtı var.
- ❌ başarısız ya da gereken şey eksik.
- ➖ uygulanmaz (örn. şema değişmedi).
- ⚠️ emin olunamadı: araç yok, izin yok ya da diff'ten karar verilemiyor. Nedenini yaz. ⚠️ hiçbir zaman ✅ sayılmaz.

## Ne zaman
Tetiklenir:
- "deploy edelim mi", "yayına hazır mı", "release öncesi", "pre-deploy check"
- "production'a çıkmadan kontrol et", "merge etmeden önce kontrol"

Tetiklenmez:
- Deploy'un genel olarak nasıl işlediğini anlatma isteği
- CI/CD'yi sıfırdan kurma
- Deploy zaten patladı ve kullanıcı log ayıklamak istiyor (hata ayıklama işi)

## Adımlar
1. Base dalı bul: `git symbolic-ref refs/remotes/origin/HEAD` çıktısından `refs/remotes/origin/` önekini at (`refs/remotes/origin/main` ise base `main`). Komut hata verirse ya da sonuç belirsizse kullanıcıya sor; tahmin etme. Sonra `git diff <base>...HEAD` ve commit edilmemiş değişiklikleri (`git status`, `git diff`) birlikte al. Neden: sadece commit'lenen kısmı okursan çalışma ağacındaki riskler kaçar.
2. AGENTS.md'yi oku: `## Komutlar`, `## Veri modelleme`, `## Güvenlik ve performans`, `## Doğrulama`. Komutları sabit yazma, oradan al. AGENTS.md yoksa kullanıcıya hangi komutların çalıştırılacağını sor.
3. Aşağıdaki 7 kontrolü değerlendir:
   1. Doğrulama: AGENTS.md Doğrulama komutlarını bu turda çalıştır, çıktıyı oku. Hepsi geçmeli. Exit 0 tek başına yetmez; "0 test çalıştı" ya da atlanan adım ❌ ya da ⚠️ olur.
   2. Env: Diff'te yeni geçen env değişken adlarını bul (örn. `process.env.X`, `os.environ["X"]`, `getenv`, `env("X")`). Her ad `.env.example` ya da README gibi bir yerde belgelenmiş olmalı. Sadece adları yaz, değerleri yazma. Belgede yoksa ❌; belge metni uydurma.
   3. Migration: Diff'te şema dosyası (ORM şeması, model tanımı, tablo DDL'i) değiştiyse ve aynı diff'te yeni bir migration dosyası yoksa ❌. Migration dosyası varsa, daha önce uygulanmış bir migration'ın düzenlenmediğini ve geri alma yolunun (down/rollback) bulunduğunu diff'ten kontrol et. Aracın down migration'ı yoksa (Prisma, Drizzle, Mongo gibi) belgelenmiş bir ters migration planı (ters migration dosyası ya da adım adım geri alma notu) da geri alma yolu sayılır ve ✅ verir. Kontrol edemiyorsan ⚠️. Şema değişikliği yoksa ➖.
   4. Secret: Sadece eklenen satırları tara, diff'i tam basma: `git diff <base>...HEAD | grep -E "^\+" | grep -nE "<desen>"` (desen çift tırnak içinde yazılır: `sk_live_|AKIA|-----BEGIN|api_key *= *[\"]|(secret|token|password) *[:=] *[\"'][^\"']{8,}`; çalışma ağacı için `git diff | grep -E "^\+" | grep -nE "<desen>"` de çalıştır). Grep çıktısını her zaman maskeleyerek al: komutun sonuna `| sed -E 's/([A-Za-z0-9_]{6})[A-Za-z0-9_+\/=-]{4,}/\1***/g'` ekle, maskesiz çıktı alma. Bulgu varsa ❌, dosya adını `git diff` başlığından al ve sadece maskeli ipucu ver (örn. `sk_live_51H…***`). Değerin tamamını hiçbir yerde yazdırma.
   5. Bağımlılık: Eklenen ya da değişen bağımlılıkları tek satır gerekçeyle listele. Manifest değiştiyse lockfile da güncellenmiş olmalı, değilse ❌. Değişiklik yoksa ➖.
   6. Rollback: Geri alma yolunu (revert, önceki sürüme dönüş, flag kapatma) ve tetikleyicisini (örn. hata oranı eşiği) anlatan bir not ara: CHANGELOG, RELEASE notları, commit mesajları (`git log <base>..HEAD`). Bulursan ✅ ve nerede olduğunu yaz. Bulamazsan kullanıcıya sor, ❌ yaz ve kısa bir not taslağı hazırla. PR açıklamasına erişemezsin; orada olabileceğini varsayma.
   7. Güvenlik ve veri erişimi: Eklenen kodu oku ve şunları ara: yetki kontrolü olmayan yeni route, handler ya da Server Action; sunucu tarafında döngü içinde veritabanı sorgusu (N+1); `limit` olmadan liste dönen yeni uç nokta ya da sorgu; string birleştirerek kurulan sorgu; temizlenmeden basılan kullanıcı HTML'i (`dangerouslySetInnerHTML`, `v-html`, `|safe`, `mark_safe`, `{!! !!}`); hata yanıtında stack trace. Yetki için aynı klasördeki mevcut handler'lara ve middleware'e de bak; koruma oradan geliyorsa ✅ say ve `dosya:satır` yaz. Giriş, sağlık kontrolü ve imzalı webhook gibi bilerek açık route bulgu değildir. Her bulguyu `dosya:satır` ile yaz ve ❌ ver. Koruma var mı yok mu kodu okuyarak çıkaramıyorsan ⚠️ ver ve nedenini yaz. Bulgu yoksa ✅ ver, kanıt olarak taranan dosyaları yaz ("3 dosya tarandı"). ➖ sadece diff'te hiç route, handler, sorgu ya da HTML basımı yoksa. Ölçüt AGENTS.md'deki `## Güvenlik ve performans` bölümüdür. Arayüz performansını (re-render, bundle boyutu, istemcideki istek şelalesi, Web Vitals) bu kontrolde değerlendirme; diff ağırlıklı olarak React arayüzüyse karar satırının altına tek satır not düş: "Arayüz performansı için `react-perf-audit` çalıştırılabilir." O skill'i kendin çalıştırma.
4. Çıktıyı ver: kompakt tablo ya da liste (madde, durum, kanıt), ardından tek karar satırı:
   - `Deploy'a hazır`: hiç ❌ ve hiç ⚠️ yok.
   - `Hazır değil: N madde ❌`: yanında önce neyin düzeltileceği (secret en başta, sonra güvenlik, Doğrulama, Migration, diğerleri).
   - `Karar verilemedi: N madde ⚠️`: ❌ yok ama emin olunamayan madde var; kullanıcıdan ne gerektiğini yaz.
5. Dur. Deploy komutunu kullanıcıya bırak.

## Bitti
- 7 kontrolün hepsinin durumu (✅/❌/➖/⚠️) ve kanıtı var.
- Karar satırı kontrollerle tutarlı: tek bir ❌ varsa "Hazır değil", ❌ yok ama ⚠️ varsa "Karar verilemedi".
- Çalıştırılamayan kontrol ✅ sayılmadı, nedeniyle birlikte ⚠️ yazıldı.
- Çıktıda hiçbir secret değeri yok.

<example>
Girdi: "yayına hazır mı? base main."
Çıktı:
| Madde | Durum | Kanıt |
|---|---|---|
| Doğrulama | ✅ | lint, typecheck, test: 3/3 geçti, 142 test |
| Env | ❌ | `PAYMENT_WEBHOOK_URL` src/api/hook.ts:12, .env.example'da yok |
| Migration | ➖ | şema dosyası değişmedi |
| Secret | ✅ | eklenen satırlarda eşleşme yok |
| Bağımlılık | ✅ | `zod` eklendi (girdi doğrulama), lockfile güncel |
| Rollback | ❌ | CHANGELOG ve commit mesajlarında not yok; taslak: "Revert PR, eski sürümü yeniden yayınla; 5xx oranı %2'yi aşarsa geri al" |
| Güvenlik ve veri erişimi | ❌ | `src/app/api/orders/route.ts:8` yeni GET listesi `limit`'siz döner |

Hazır değil: 3 madde ❌. Önce siparişler listesine `limit` ekle, sonra `PAYMENT_WEBHOOK_URL` adını .env.example'a ekle, en son rollback notunu CHANGELOG'a yaz.
</example>

## Yapma
- Deploy, push, publish, production migration ya da release komutu çalıştırma.
- Secret değerini yazdırma (çıktıda, alıntıda, log özetinde).
- Çıktıyı okumadan exit 0'a bakıp "doğrulandı" deme.
- Kanıtsız ✅ verme; kanıt bulunamıyorsa ⚠️ ya da ❌ yaz.
- Olmayan env belgesi uydurma ya da belgelenmemiş değişkeni "belgelenmiş" sayma.
