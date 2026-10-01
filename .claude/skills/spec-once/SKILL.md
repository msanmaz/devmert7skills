---
name: spec-once
description: Basit olmayan işlerde tahmin yürütmeyi bırakır. Önce kodu ve AGENTS.md'yi okur, sadece karar gerektiren noktaları 3-5 numaralı soruyla (her birinde önerilen cevap) tek mesajda sorar, cevaplardan kısa bir spec yazar (hedef, kapsam, kapsam dışı, kabul kriterleri, dokunulacak dosyalar, doğrulama) ve açık onay almadan tek satır kod yazmaz. Spec sohbette yazılır, dosya açılmaz. Şu durumlarda kullan: istek belirsiz ya da açık uçluysa ya da iş en az orta boydaysa; örnek istekler "kullanıcılar için bildirim sistemi ekle", "ödeme akışını yeniden yap", "dashboard'a rapor ekle", "refactor the auth flow", "spec yaz". Şu durumlarda kullanma: plan modu açıkken (kendi planlama akışı var), yazım hatası, tek satırlık düzeltme, yeniden adlandırma, tek cümleyle anlatılabilen ve uygulaması belli küçük görevler ("README'deki yazım hatasını düzelt").
---

# spec-once

## Amaç

Basit olmayan işte önce soruları ve spec'i onaylat, sonra kodla; çünkü yanlış varsayımla yazılan kod sonradan çöpe gider.

## Ne zaman

Tetiklenir:
- İstek belirsiz ya da açık uçlu ("dashboard'a rapor ekle").
- İş en az orta boyda ("kullanıcılar için bildirim sistemi ekle", "ödeme akışını yeniden yap").
- "Yeni özellik ekle" gibi ifadeler tek başına tetik değildir; belirsizlik ya da boyut şarttır.

Tetiklenmez:
- Plan modu açıkken. Plan modunun kendi planlama akışı var; iki onay akışını üst üste bindirme.
- Yazım hatası, tek satırlık düzeltme, yeniden adlandırma.
- Bir cümleyle anlatılabilen ve tek bariz uygulaması olan iş ("README'deki yazım hatasını düzelt").
- Kapsamı zaten net yazılmış küçük görev.

## Adımlar

1. İşi tek satırda boyutlandır: küçük / orta / büyük. Küçük ve netse bu skill'i bırak ve işi doğrudan yap; spec yazmak orada sadece gecikme getirir.
2. İlgili kodu ve AGENTS.md'yi oku. Kodun cevaplayabildiği hiçbir şeyi sorma (mevcut yapı, isimlendirme, kullanılan kütüphane, test düzeni). Kullanıcının vakti sadece karar için harcanmalı.
3. Yapı kontrolü yap: yeni iş mevcut yapıya doğal oturuyor mu? Arayüz dışı işte "component" yerine modül ya da fonksiyon oku. Şunlar uyumsuzluk işaretidir: dokunacağın component ya da modül zaten birden fazla iş yapıyor; ihtiyaç duyulan veri yanlış katmanda ve birkaç kat prop/parametre ile taşınması gerekiyor; ortak bir parçaya sadece bu iş için `if` ya da özel seçenek eklemek gerekiyor. Uyumsuzluk varsa spec'e davranışı değiştirmeyen bir "Önce hazırlık" adımı koy. Bu adım ayrı bir değişiklik olarak asıl işten önce yapılır (commit ya da PR kararı kullanıcıdadır); yapı yeni işe uyunca asıl değişiklik de küçülür. Uyumsuzluk yoksa bunu da açıkça yaz: "Hazırlık gerekmiyor."
4. Tek mesajda 3-5 numaralı soru sor. Her sorunun yanında önerilen bir varsayılan cevap yaz ki kullanıcı "hepsi önerdiğin gibi" diyebilsin. Sadece kararları sor: davranış, kapsam, uç durumlar, ödünleşimler. Soruları tek tek damlatma; her tur kullanıcıyı bekletir.
5. Cevaplar gelince kısa spec'i sohbette yaz (en fazla 30 satır; kullanıcı açıkça istemedikçe dosya oluşturma):
   - Hedef: bir cümle.
   - Kapsam / Kapsam dışı: kısa maddeler; dışarıda bırakılanı açıkça yaz, çünkü kapsam kayması en sık buradan başlar.
   - Kabul kriterleri: her biri tek tek test edilebilir olsun ("kullanıcı X yaptığında Y görünür", "boş girdide hata döner"). Kimin görüp değiştirebileceği (yetki) ve liste varsa sayfalama da birer kriterdir.
   - Önce hazırlık: 3. adımdaki sonuç; hazırlık adımları ya da "gerekmiyor".
   - Bileşenler (sadece arayüz işinde): her yeni component için tek satır: veriyi kim yükler, hangi etkileşimin sahibi, yükleniyor / boş / hata hali. En fazla 5 satır; fazlasını gruplayıp tek satıra indir. Veriyi yükleyen ya da etkileşimin sahibi belli değilse component bölünmesi yanlıştır, onu düzelt.
   - Dokunulacak dosyalar: koddan okuduğun gerçek yollar.
   - Doğrulama: AGENTS.md'deki `## Doğrulama` ve `## Komutlar` bölümlerindeki komutlar artı bu özelliğe özel bir uçtan uca kontrol. Komut uydurma. AGENTS.md yoksa kullanıcıya hangi komutları çalıştırması gerektiğini sor.
6. Dur ve açık onay bekle. Spec'le aynı mesajda kodlamaya başlamak onayı atlamak sayılır; bu skill'in ana kuralı budur. "Olur", "devam", "onaylıyorum" onaydır; sessizlik veya "hmm" değildir. Kullanıcı değişiklik isterse spec'i güncelle ve yine onay bekle. Spec'teki "Önce hazırlık" adımı da bu onaya dahildir; spec'te yoksa yapma.
7. Onaydan sonra kabul kriterlerine göre uygula, sonunda Doğrulama adımlarını çalıştır ve sonucu prove-it şablonuyla raporla: her kabul kriteri bir kontrol listesi maddesidir, 6'dan fazlaysa aynı durumdakileri birleştir.

## Bitti

- Kullanıcı, her kabul kriteri ayrı ayrı kontrol edilebilen bir spec'i onayladı.
- Spec'te yapı kontrolünün sonucu yazılı: hazırlık adımları ya da "gerekmiyor".
- Onaydan önce hiçbir kod yazılmadı, hiçbir dosya değiştirilmedi.
- Uygulama sonrası her kriter için komut çıktısı veya gözlem var.

## Yapma

- Tek soru sorup cevap bekleyip sonra bir soru daha sorma (damlatma).
- 5'ten fazla soru sorma; fazlası kararı değil kullanıcıyı yorar.
- Repoda bulunabilecek bilgiyi sorma.
- 30 satırı aşan spec yazma; uzun spec'i okumak zorlaşır ve onay gerçek bir kontrol olmaktan çıkar.
- Onay gelmeden kod yazma veya dosya düzenleme.
- Kullanıcı istemeden spec'i dosyaya yazma.
- Küçük ve net işte bu akışı çalıştırma.
- Hazırlık adımını asıl işle aynı değişikliğe karıştırma; davranış değiştirmeyen düzenleme ayrı gider.

<example>
İstek: "kullanıcılar için bildirim sistemi ekle"

Boyut: büyük (veri modeli, API ve arayüz). Kodu ve AGENTS.md'yi okudum; projede zaten e-posta gönderen bir kuyruk var, onu sormuyorum.

1. Hangi kanallar? Önerim: sadece uygulama içi bildirim, e-posta sonra.
2. Hangi olaylar bildirim üretsin? Önerim: yorum, mention ve sipariş durumu değişikliği.
3. Okundu bilgisi saklansın mı ve ne kadar süre? Önerim: saklansın, 90 gün sonra silinsin.

Cevaplayınca spec'i burada yazıp onayını bekleyeceğim; ondan önce kod yok.
</example>
