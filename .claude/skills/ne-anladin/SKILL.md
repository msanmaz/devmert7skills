---
name: ne-anladin
description: Kod yazmadan önce kullanıcının isteğini kendi kelimeleriyle geri söyler; hedefi, çözülmek istenen problemi, istekte yazmadığı için ajanın kendi seçeceği kararları ve kapsam dışını yazar. Kodda isteği boşa çıkaracak bir şey görürse onu da söyler, sonra durup onay bekler; dosya değiştirmez. Şu durumlarda kullan: kullanıcı ajanın kendisini doğru anlayıp anlamadığını kontrol etmek istediğinde; örnek ifadeler "ne anladın?", "beni doğru anladın mı?", "önce ne anladığını anlat", "kendi kelimelerinle anlat", "başlamadan anladığını söyle", "in your own words", "what did you understand?", "/ne-anladin". Şu durumlarda kullanma: kullanıcı bunu istemeden verilen sıradan işler (belirsiz ya da büyük işte soru ve spec için spec-once var), iş bittikten sonraki "bitti mi?" kontrolü, kodu ya da bir kavramı açıklatan sorular ("bu fonksiyon ne yapıyor?").
---

# ne-anladin

## Amaç

Kısa bir istek kararları açık bırakır ve ajan onları sormadan kendisi kapatır; bu skill o kararları kod yazılmadan önce kullanıcıya geri getirir, çünkü yanlış anlaşılmayı koddan önce düzeltmek yazılmış kodu geri almaktan ucuzdur.

## Ne zaman

Tetiklenir:
- Kullanıcı açıkça ajanın kendisini nasıl anladığını soruyor: "ne anladın?", "beni doğru anladın mı?", "başlamadan anladığını söyle", "/ne-anladin".
- Soru istekle aynı mesajda olabilir ("ne anladın? arama kutusu ekle") ya da istekten sonra ayrı bir mesajda gelebilir. Ayrı mesajsa sohbetteki son isteği geri söyle.

Tetiklenmez:
- Kullanıcı bunu istemedi. Sıradan isteği doğrudan yap; istek belirsiz ya da büyükse spec-once'un işi.
- İş bitti ve kullanıcı sonucu soruyor ("bitti mi?"): prove-it'in işi.
- Kodu ya da bir kavramı açıklatan soru ("bu fonksiyon ne yapıyor?").

## Adımlar

1. Gerekiyorsa ilgili dosyaları oku; sadece oku. Dosya değiştirme, paket kurma, durum değiştiren komut çalıştırma. Koddan okunabilen şeyi (dosya düzeni, kullanılan kütüphane) kullanıcıya sorma.
2. Şu dört başlıkla, kendi kelimelerinle yaz:
   - **Hedefin:** tek cümle. Kullanıcının sonunda elde etmek istediği durum, istediği özelliğin adı değil ("bildirim ekle" → "yapılmamış işler hatırlatılsın").
   - **Çözmeye çalıştığın problem:** tek cümle, bu istek neden şimdi geldi. İstekte yazmıyorsa tahminini yaz ve sonuna "(tahmin)" ekle.
   - **Senin yerine vereceğim kararlar:** istekte yazmayan ama işi yaparken seçmek zorunda kalacağın şeyler; en fazla 5, en önemlisi önce. Her birinde ne seçeceğini yaz ("Saat: her gün 20:00"), çünkü kullanıcı somut bir seçime açık bir sorudan daha hızlı itiraz eder.
   - **Yapmayacaklarım:** kullanıcının bekleyebileceği ama kapsam dışında bıraktığın şeyler. Yoksa "yok" yaz.
3. Kodda isteği boşa çıkaracak ya da sonucunu yanlış yapacak bir şey gördüysen ayrı bir satırda yaz: "Dikkat: ..." ve neden önemli olduğunu ekle. Koddan gösteremediğin bir sorunu yazma.
4. İş birden fazla parçaya, veri modeli değişikliğine ya da belirsiz bir kapsama dönüşüyorsa, kararlardan sonra spec-once ile devam etmeyi öner.
5. Mesajı tek bir soruyla bitir: "Doğru anladım mı? Değiştirmek istediğin karar var mı?" Sonra dur. Aynı mesajda işe başlamak, kullanıcının düzeltme fırsatını elinden alır; bu skill'in ana kuralı budur.
6. Kullanıcı onaylarsa ya da bazı kararları değiştirirse ("saat 21 olsun, gerisi tamam") değiştirdiği kararlarla işe başla, dört başlığı yeniden yazma. Hedef ya da problem değiştiyse dört başlığı yeniden yaz ve tekrar sor.

## Bitti

- Mesajda dört başlık var ve her kararın yanında seçilen değer yazıyor.
- "Dikkat" satırı yalnızca koddan gösterilebilen bir sorun için var.
- Mesaj soruyla bitiyor; hiçbir dosya değişmedi, kod yazılmadı.
- Mesaj 15 satırı geçmiyor.

## Yapma

- Kullanıcının cümlesini kelimesi kelimesine tekrarlama; tekrar, anladığını göstermez.
- Plan, adım listesi ya da kod yazma. Bu adımın sorusu "ne ve neden", "nasıl" değil.
- Kararları açık uçlu soruya çevirme ("Saat kaç olsun?"); kullanıcıyı ikinci bir tura sokar.
- Listeyi önemsiz kararlarla doldurma (değişken adı, dosya adı).

<example>
İstek: "ne anladın? bu uygulamaya bildirim ekle"

**Hedefin:** Yapılmamış alışkanlıklar gün içinde kendiliğinden hatırlatılsın.
**Çözmeye çalıştığın problem:** Sayfaya bakmayınca alışkanlıkları unutuyorsun. (tahmin)
**Senin yerine vereceğim kararlar:**
1. Ne zaman: her gün 20:00, sayfadan değiştirilebilir.
2. Hangi durumda: sadece sekme açıkken. Kapalıyken de gelmesi push bildirimi ve sunucu ister; projede sunucu yok.
3. Hangi alışkanlıklar: o gün işaretlenmemiş olanlar.
**Yapmayacaklarım:** e-posta ve telefon bildirimi.
Dikkat: `done` işaretleri gün değişince sıfırlanmıyor; dün yapılan alışkanlık bugün de yapılmış görünür ve hatırlatmaya girmez. Önce günlük sıfırlamayı eklemeyi öneririm.

Doğru anladım mı? Değiştirmek istediğin karar var mı?
</example>
