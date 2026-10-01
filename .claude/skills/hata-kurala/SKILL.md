---
name: hata-kurala
description: Kullanıcı ajanı tek bir düzeltmeyle ("hayır, burada console.log yerine logger kullan") ya da tekrarlı hatırlatmayla düzelttiğinde devreye girer; işi düzeltir ve AGENTS.md'ye tek satırlık, nedenli bir kural eklemeyi teklif eder. Aynı anlamda kural varsa kopya eklemez. Kural istenir ya da hata tekrarlanırsa öneriyi her zaman unified diff olarak gösterir, eksik bilgiyi diff'in altında sorar, onay almadan yazmaz. Şu durumlarda kullan: "hayır, X yerine Y kullan", "yine aynı hatayı yaptın", "bir daha böyle yapma", "kural olarak ekle", "bunu AGENTS.md'ye yaz", "don't do that again", "add this as a rule". Şu durumlarda kullanma: sadece bu görev için geçerli tercihler, yazım hatası düzeltmeleri, sorular.
---

# hata-kurala

## Amaç

Tekrarlayan ya da kural olarak istenen düzeltmeyi repoya özgü tek satırlık bir AGENTS.md kuralına çevir ve onay almadan hiçbir şey yazma; çünkü onaysız yazılan kural dosyayı kirletir ve kullanıcının kontrolünü alır.

## Ne zaman

Tam akış (adım 1-7) şu durumlarda çalışır:
- Aynı hatırlatma ikinci kez geliyor ("yine aynı hatayı yaptın", "bir daha böyle yapma").
- Kullanıcı açıkça kural istiyor ("kural olarak ekle", "AGENTS.md'ye yaz").
- Kullanıcı ajanın değişikliğini geri alıyor ve nedeni bir alışkanlık hatası.

Tek bir olağan düzeltmede ("hayır öyle değil", "bunu yapma"): önce işi düzelt. Sonra yalnızca şunu sor: "Bunu AGENTS.md'ye kural olarak ekleyeyim mi?" Taslak satır ya da diff yazma; kullanıcı evet derse tam akışı başlat. Sebep: tek seferlik düzeltmelerin çoğu kural olmaz, her biri için taslak üretmek kullanıcıyı yorar.

Tetiklenmez:
- Sadece bu görev için geçerli tercih ("bu sefer tablo yerine liste kullan").
- Kullanıcının kendi mesajındaki yazım hatası düzeltmesi.
- Soru ("neden böyle yaptın?"). Önce soruyu yanıtla; yanıt gerçek bir hatayı ortaya çıkarırsa ve kullanıcı kural isterse bu skill'e dön.

## Adımlar

1. Hatayı adlandır. Tek cümle, iki parçalı: önce ajanın gerçekte yaptığı, sonra kullanıcının beklediği. Örnek: "Yapılan: `../` ile relative import. Beklenen: `@/` alias." Parçaları ters yazma. Cümleyi kuramıyorsan düzeltme tek seferlik bir tercihtir; dur ve kural önermeden devam et.
2. Mevcut kuralı ara. AGENTS.md'yi oku ve hatayı kapsayan satırı anlamına göre ara, sadece birebir kelimeyle değil. Bulursan yeni satır ekleme, çünkü kopya kurallar dosyayı şişirir ve çelişki doğurur. İki seçenek sun:
   - Satır var ama muğlak: o satırı netleştirmeyi öner (aynı satırın yeni hali).
   - Satır net: "Bu kural zaten var, sorun uyum" de, satırı göster, bir sonraki adımda neyin farklı yapılacağını söyle.
3. Bölümü seç. Mevcut başlıklardan biri: kod kuralları, veri modelleme, test stratejisi, kritik kurallar ya da doğrulama. Komutlar bölümüne sadece yanlış ya da eksik komut varsa dokun. Yeni bölüm açma.
4. Tek satır yaz. Biçim: talimat + tek cümle neden. Bu repoya özgü olsun (gerçek dizin, araç, komut adı). "Ne yap" biçiminde yaz, mümkünse "yapma" demeden. "Dikkatli ol", "temiz kod yaz" gibi genel tavsiye kural değildir. Görevin ayrıntısını (dosya adı, ticket, tarih) satıra taşıma.
5. Mekanik mi diye bak. Hata biçim, import sırası, adlandırma gibi lint ile yakalanabilen türdense, yazılı kuralın yanında ya da yerine lint, CI ya da hook kontrolü öner. Çünkü AGENTS.md tavsiye niteliğindedir, araç ise her seferinde zorlar. Kontrolü kurma, sadece öner.
6. Diff göster. AGENTS.md için her zaman unified diff ver; bilgi eksik olsa bile (örn. bölüm ya da dizin belirsiz) en iyi tahminle satırı diff olarak yaz ve eksik bilgiyi diff'in altında sor. Düz metin taslak verme. Değişiklik tam bir satır olsun (yeni ya da netleştirilmiş). Dosya bu satırla 200 satırı aşacaksa: bir satır ekle, gerekirse en zayıf bir satırı çıkar (diff'te `-` ile göster, nedenini yaz); toplam yine bir ekleme ve en fazla bir çıkarma olsun. Satır sayısını `wc -l` ile gerçekten say.
7. Onay bekle. Kullanıcı açıkça onaylamadan yazma. "Tamam", "ekle", "olur" onaydır; sessizlik ya da konuyu değiştirmek değildir. Onay gelince gerçek dosyayı (AGENTS.md) düzenle. CLAUDE.md symlink olabilir; her zaman hedef dosya AGENTS.md'yi düzenle.

AGENTS.md yoksa kullanıcıya nerede tutulduğunu sor; kendi başına dosya oluşturma. Doğrulama komutu gerekiyorsa AGENTS.md'deki `## Doğrulama` ve `## Komutlar` bölümlerinden oku, komut uydurma.

## Bitti

- AGENTS.md'de, kullanıcının onayladığı tam olarak bir yeni ya da netleştirilmiş satır var; 200 satır aşılıyorsa en zayıf bir satır da çıkarıldı (ya da kullanıcı reddetti ve dosya değişmedi).
- Aynı anlama gelen ikinci bir satır yok (yeniden okuyup kontrol et).
- `wc -l AGENTS.md` sonucu 200 ya da altı.
- CLAUDE.md'ye dokunulmadı.
- Kullanıcıya eklenen satır ve varsa önerilen lint/CI kontrolü bildirildi.

Kullanıcı reddederse ya da kural zaten varsa dosya değişmeden biter; bu da geçerli bir sonuçtur.

## Yapma

- Onaysız yazma. Bu skill'in tek vurgulu kuralı budur.
- Tek bir olağan düzeltmede taslak satır ya da diff üretme; sadece kural ekleme teklifi yap.
- Genel tavsiye ekleme ("dikkatli ol", "testleri unutma").
- Tek seferlik görev ayrıntısını kurala çevirme.
- Bir hata için birden fazla satır ekleme.
- Var olan kuralı tekrarlama; netleştir ya da uyum sorunu olduğunu söyle.
- CLAUDE.md'yi düzenleme.

<example>
Kullanıcı: "Yine servis içinde ham SQL yazdın. Sorgular sadece `db/queries/` altında olsun, bir daha böyle yapma."

1. Hata: sorgu `services/order.py` içine yazıldı, `db/queries/` bekleniyordu. "Yine" dediği için tam akış çalışır.
2. AGENTS.md'de sorgu konumu hakkında satır yok.
3. Bölüm: Kod kuralları. Mekanik değil, lint önerisi yok.
4. Satır: "SQL sorgularını sadece `db/queries/` altına yaz. Sorgular tek yerde olunca şema değişikliğinde hepsi bulunur."

```diff
--- a/AGENTS.md
+++ b/AGENTS.md
@@ -12,1 +12,2 @@ ## Kod kuralları
 - Tip ipucu olmayan fonksiyon ekleme. Araçlar imzaya güvenir.
+- SQL sorgularını sadece `db/queries/` altına yaz. Sorgular tek yerde olunca şema değişikliğinde hepsi bulunur.
```

Dosya 112 satır, sınır içinde. Ekleyeyim mi?
</example>
