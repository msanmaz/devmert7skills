# devmert7-agents

Kodlama ajanın için kurallar ve 5 skill. Claude Code, Codex, Cursor ve Copilot ile çalışır.

```
devmert7-agents/
├── AGENTS.md                  stack'ine göre üretilmiş kurallar, Doğrulama bölümlü
├── CLAUDE.md -> AGENTS.md     tek dosya, bütün ajanlar
└── .claude/skills/
    ├── prove-it/              yapılanı sade, kanıtlı ve kontrol edilebilir listeye çevirir (her iş için)
    ├── hata-kurala/           düzelttiğin hatayı AGENTS.md'ye tek satır kural yapar
    ├── spec-once/             büyük işte kod yazmadan önce soru sorar, spec yazar
    └── deploy-oncesi/         deploy öncesi kanıtlı kontrol listesi, deploy yapmaz
```

## Kurulum

1. Sitedeki oluşturucudan kendi stack'in için `AGENTS.md` dosyasını indir ve repo köküne koy. Bu klasördeki `AGENTS.md` sadece örnektir (TypeScript, Next.js, pnpm, Vitest, PostgreSQL + Drizzle).
2. `.claude/skills/` klasörünü repo köküne kopyala.
3. Claude Code için CLAUDE.md'yi bağla:
   ```bash
   ln -s AGENTS.md CLAUDE.md
   ```
   Windows'ta symlink klonlarda bozulur. Onun yerine tek satırlık bir `CLAUDE.md` yaz:
   ```
   @AGENTS.md
   ```
   İkisi de aynı işi görür: Claude Code kuralları AGENTS.md'den okur. Codex, Cursor ve Copilot AGENTS.md'yi doğrudan okur.

## Kullanım

- Skill'ler kendiliğinden devreye girer. Elle çağırmak için: `/prove-it`, `/hata-kurala`, `/spec-once`, `/deploy-oncesi`, `/react-perf-audit`.
- Hangisi ne zaman:
  | Durum | Skill |
  |---|---|
  | Ajan "bitti" dedi ya da yaptığı işi uzun bir metinle anlattı (kod, araştırma, yazı, fark etmez) | prove-it |
  | Ajan aynı hatayı ikinci kez yaptı | hata-kurala |
  | İş büyük ya da belirsiz | spec-once |
  | "Canlıya alalım mı?" | deploy-oncesi: güvenlik, migration, secret, sunucu tarafı sorgular |
  | "Bu sayfa neden yavaş?" | react-perf-audit: re-render, istemci istek şelalesi (N+1), bundle, Core Web Vitals |
- AGENTS.md'deki `## Güvenlik ve performans` kuralları yavaşlığı baştan önler; react-perf-audit kaçanı bulur. Dev server açıksa ve Chrome DevTools MCP kuruluysa Web Vitals'ı gerçekten ölçer, değilse o kısmı atladığını söyler. Kodu değiştirmez; düzeltmeyi rapordan sonra ayrıca istersin.
- Kritik davranış skill'e bağlı değil: Doğrulama kuralları AGENTS.md'de de yazılı, skill tetiklenmese de ajan komutları çalıştırır.
- AGENTS.md yaşayan bir dosyadır. Ajan bir hatayı iki kez yaparsa `hata-kurala` o hatayı tek satır kurala çevirir, sen onaylarsın.

## Dosyayı kısa tut

AGENTS.md 200 satırın altında kalsın (Claude Code'un önerisi). Her satır için sor: "Bunu silersem ajan hata yapar mı?" Yapmıyorsa sil.

## İlham

Bu paket, obra/superpowers, anthropics/claude-plugins-official, mattpocock/skills ve Vercel'in doğrulama sözleşmesinden fikir aldı. Yalnızca fikirler alındı, metinler yeniden yazıldı.
