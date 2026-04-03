# Agent Intelligence — Veri → Anlam → Aksiyon Haritası

## Hook Sisteminden Çıkarılabilecek Ham Veriler

Context-mode 4 hook üzerinden veri topluyor. Her hook'un ürettiği ham veri:

### PostToolUse (her tool call sonrası)
- Tool adı (Bash, Read, Write, Edit, Grep, Glob, Agent, Task, WebFetch)
- Tool argümanları (dosya yolu, komut, arama sorgusu)
- Sonuç (stdout, stderr, exit code)
- Session ID, timestamp, proje dizini

### PreToolUse (her tool call öncesi)
- Tool adı ve argümanları (henüz çalışmadan)
- Deny/modify kararı
- Routing bilgisi

### SessionStart (session yaşam döngüsü)
- Kaynak: startup / compact / resume / clear
- Session ID, proje dizini
- Compact sayısı (context doluluk proxy'si)

### UserPromptSubmit (kullanıcı prompt'ları)
- Prompt metni
- Timestamp
- Kullanıcı kararları ve düzeltmeleri

---

## Personas

### Persona 1: VP Engineering / CTO
**Bütçe sahibi. Board'a rapor veriyor.**

### Persona 2: Engineering Manager
**Takım performansından sorumlu. Sprint'leri planlıyor, 1:1 yapıyor.**

### Persona 3: Platform / DevEx Lead
**Tool standardizasyonu, developer experience, onboarding.**

### Persona 4: Developer (dolaylı)
**Alıcı değil ama veri kaynağı. Kendi workflow'unu optimize etmek istiyor.**

---

## Veri → Anlam → Aksiyon Haritası

### VP Engineering / CTO

| Ham Veri | Nasıl Hesaplanır | Anlam | Çözdüğü Sorun | Aksiyon |
|----------|-----------------|-------|---------------|---------|
| Session sayısı × developer | SessionStart event count per user | Gerçek AI tool kullanım oranı | "184 Copilot lisansı aldık ama kaçı kullanıyor?" | Kullanılmayan lisansları iptal et, yıllık $85K tasarruf |
| Tool call dağılımı (Cursor vs Claude vs Copilot) | PostToolUse tool name grupla | Hangi tool gerçekten kullanılıyor | "3 farklı AI tool'a para veriyoruz, hangisi dominant?" | Tek tool'a consolidate et, vendor negotiation leverage |
| Session başına ortalama commit | PostToolUse'da git commit detect / session count | AI session'larının deliverable üretme oranı | "AI kullanımı arttı ama output artmadı" | Productive vs. unproductive AI usage'ı ayırt et |
| Haftalık aktif AI kullanıcı trendi | Unique session ID per week | Adoption eğrisi yükseliyor mu düşüyor mu | "AI adoption'ı mandate ettik ama gerçekten oluyor mu?" | Adoption düşüyorsa: eğitim/tooling yatırımı. Artıyorsa: budget justify |
| Compact sayısı / session | PreCompact event count | Karmaşık iş mi yapılıyor, yoksa basit autocomplete mi | "AI'ı sadece boilerplate için mi kullanıyoruz?" | Yüksek compact = karmaşık iş. Düşük = sadece snippet. Stratejiyi ayarla |
| Error rate per tool | PostToolUse exit code != 0 / total calls | Hangi tool en çok hata üretiyor | "Cursor'ın code gen kalitesi Copilot'tan düşük mü?" | Tool karşılaştırma verisiyle procurement kararı |

**Board'a sunulacak single metric**: AI ROI Score = (AI-assisted commit count × avg session efficiency) / monthly AI spend

---

### Engineering Manager

| Ham Veri | Nasıl Hesaplanır | Anlam | Çözdüğü Sorun | Aksiyon |
|----------|-----------------|-------|---------------|---------|
| Developer bazlı session süresi | SessionStart → son PostToolUse timestamp farkı | Kim uzun session'larda takılıyor, kim hızlı bitiriyor | "Ahmet her task'a 3 saat harcıyor, Ayşe 45 dakikada bitiriyor" | Ahmet'le 1:1'de AI workflow coaching yap |
| Edit→Test→Edit döngü sayısı | PostToolUse'da Write/Edit → Bash(test) → Write/Edit pattern detect | Iteration verimliliği — kaç deneme ile çözüme ulaşıyor | "Junior'lar 12 iterasyonda çözüyor, senior'lar 3'te" | Junior'lara TDD eğitimi ver, prompt engineering workshop |
| Dosya dokunma dağılımı per developer | PostToolUse Read/Write/Edit file path extract | Kim hangi alan üzerinde çalışıyor | "Sprint'te API task'ı var ama developer frontend dosyalarına dokunuyor" | Sprint alignment problemi — task reassign veya pairing |
| İlk commit'e kadar geçen süre (per session) | SessionStart → ilk git commit PostToolUse arası | Time-to-first-delivery | "Yeni feature branch'te 2 gün commit yok" | Blocker var mı? Scope büyük mü? Kır |
| Rework rate | Aynı dosyanın aynı session'da N kez Edit edilmesi | Kararsızlık veya yanlış yöne gitme | "Bu PR'da 47 edit var aynı dosyaya — ne oluyor?" | Code review öncesi pairing, spec netleştirme |
| UserPromptSubmit'te "fix", "error", "bug" frekansı | Prompt metin analizi | Developer kaç kez AI'dan fix istiyor vs. yeni feature | "Session'ın %70'i hata düzeltme — kök neden ne?" | Technical debt sprint'i planla |
| Exploration/Execution oranı | (Read+Grep+Glob) / (Write+Edit) | Araştırma mı yapıyor, üretim mi | "Çok okuyor ama az yazıyor — onboarding mi, takılma mı?" | Yeni developer: normal. 6 aylık developer: codebase docs geliştir |

**Sprint retro metric**: Team AI Effectiveness = (committed features / AI session hours) × (1 - rework rate)

---

### Platform / DevEx Lead

| Ham Veri | Nasıl Hesaplanır | Anlam | Çözdüğü Sorun | Aksiyon |
|----------|-----------------|-------|---------------|---------|
| Tool adoption curve per developer | İlk kullanım → düzenli kullanım kaç gün sürüyor | Onboarding friction | "Context-mode'u kurduk ama %40'ı 3 gün sonra bıraktı" | Onboarding guide iyileştir, default config sağla |
| ctx_execute vs Bash kullanım oranı | PostToolUse tool name karşılaştır | Sandbox routing'in adoption'ı | "Developer'lar hala Bash'te 500 satırlık output dump'lıyor" | Routing block'u güçlendir, eğitim materyali |
| MCP tool keşif sırası | Her developer'ın ilk kez hangi tool'ları kullandığı timeline | Tool discovery path | "Kimse ctx_search kullanmıyor, hep ctx_execute" | Tool documentation ve discovery UX iyileştir |
| Yeni developer'ın ilk hafta metrikleri vs. 3. ay | Session count, tool diversity, error rate trend | Onboarding velocity | "Yeni hire'lar 6 haftada diğerlerinin %80'ine ulaşıyor" | Onboarding programını 6→3 haftaya kısalt |
| En çok kullanılan prompt pattern'leri | UserPromptSubmit text clustering | Tekrarlayan iş pattern'leri | "Herkes aynı boilerplate'i istiyor" | Skill/snippet/template oluştur, otomate et |
| Platform bazlı session başarı oranı | Commit ile biten session / toplam session, per platform | Hangi IDE/tool combo en verimli | "Cursor + context-mode %78 commit rate, Copilot + context-mode %52" | Tool standardizasyon kararı |

**DevEx north star**: Time-to-Productive = yeni developer'ın org ortalamasının %80'ine ulaşma süresi

---

## Hook'lardan ÇIKARILABİLECEK Metrikler (kesin)

1. **Session sayısı / süre / frekans** — SessionStart
2. **Tool call dağılımı** — PostToolUse
3. **Error rate** — PostToolUse exit code
4. **Dosya dokunma haritası** — PostToolUse file args
5. **Git operasyonları** — PostToolUse Bash(git *)
6. **Edit-Test-Edit döngüsü** — PostToolUse pattern matching
7. **Compact frekansı** — PreCompact event count
8. **Context efficiency** — bytes saved / total processed
9. **Prompt pattern'leri** — UserPromptSubmit text
10. **Kullanıcı kararları** — UserPromptSubmit corrections
11. **Session outcome** — son tool call'ın commit olup olmadığı
12. **Rework rate** — aynı dosya N kez edit
13. **Exploration/execution ratio** — Read+Grep / Write+Edit
14. **Tool discovery timeline** — developer bazlı ilk kullanım tarihleri
15. **Cross-platform kullanım** — adapter detect per session

## Hook'lardan ÇIKARILAMAYACAK Metrikler (ek veri kaynağı gerekir)

1. **Kod kalitesi** — linter/SonarQube entegrasyonu gerekir
2. **PR review süresi** — GitHub API gerekir
3. **Business impact** — product analytics gerekir
4. **Developer memnuniyeti** — survey gerekir
5. **Gerçek zaman tasarrufu** — kontrol grubu (AI'sız) gerekir
6. **Deployment frequency** — CI/CD pipeline entegrasyonu gerekir

---

## VC Sorusu Özeti

> "Hangi veri, hangi anlam taşıyor ve hangi sorunu çözerek o pozisyonda çalışan kişiyi nasıl aksiyona geçiriyor?"

| Persona | #1 Verisi | Anlamı | Aksiyonu |
|---------|----------|--------|---------|
| **CTO** | Aktif AI kullanıcı / toplam lisans | Gerçek adoption rate | Kullanılmayan lisansları kes → yıllık $85K+ tasarruf |
| **CTO** | Session başına commit oranı | AI'ın productive kullanılıp kullanılmadığı | Low-output team'lere müdahale veya tool değiştir |
| **EM** | Developer bazlı iteration count | Kimin coaching'e ihtiyacı var | 1:1'de AI workflow training |
| **EM** | Error rate per developer | Kimin kalite problemi var | Code review pairing, TDD eğitimi |
| **DevEx** | Tool adoption curve | Onboarding friction nerede | Docs/config/template iyileştir |
| **DevEx** | Prompt pattern clustering | Tekrarlayan manuel iş | Otomasyon/skill/template oluştur |
