# 🔄 ALEX GİT ÇALIŞMA KURALLARI
**Investment Hunter v2.0 - Git-Based Development Protocol**

## 🎯 **TEMEL ÇALIŞMA PRENSİPLERİ**

### **Git Workflow Kuralları**
```bash
BRANCH STRATEJİSİ:
main (Production-ready, her zaman çalışır)
├── develop (Integration branch, test edilmiş özellikler)
├── feature/[task-id]-[short-description] (Her görev ayrı branch)
├── hotfix/[issue-description] (Kritik düzeltmeler)
└── release/v[version] (Release hazırlıkları)

BRANCH NAMING CONVENTION:
feature/1.1.1-database-connection
feature/1.2.3-n8n-webhook-integration  
feature/2.1.1-advanced-search-filters
hotfix/api-timeout-fix
```

### **Günlük Çalışma Döngüsü**
```bash
1. GÖREV BAŞLATMA:
   - Progress tracker'dan aktif görevi kontrol et
   - Yeni feature branch oluştur: git checkout -b feature/[task-id]-[name]
   - Sadece o görevle ilgili değişiklik yap

2. GELIŞTIRME:
   - Küçük, anlamlı commit'ler yap
   - Her commit'te açıklayıcı mesaj kullan
   - Test et: npm run build && npm run dev

3. TAMAMLAMA:
   - Final test yap (build + manual test)
   - develop branch'e merge et
   - Branch'i sil: git branch -d feature/[name]
   - Progress tracker'ı güncelle

4. RAPORLAMA:
   - Her görev sonrası durum raporu ver
   - Sonraki görevi sor
   - Sorun varsa hemen bildir
```

### **Commit Message Formatı**
```bash
CONVENTIONAL COMMITS:
feat: add PostgreSQL database connection
fix: resolve API timeout issue
perf: optimize database queries
docs: update API documentation
style: fix code formatting
refactor: restructure component hierarchy
test: add unit tests for API calls

DETAYLI COMMIT ÖRNEĞİ:
feat: implement real-time price updates via n8n webhooks

- Add webhook endpoint for n8n integration
- Implement real-time data sync mechanism
- Add price change notification system
- Update UI to show live price changes
- Add error handling for webhook failures

Tested: ✅ Build successful, webhooks working
Related: Task 1.4.1 - n8n webhook integration
```

### **Merge ve Test Protokolü**
```bash
HER MERGE ÖNCESİ ZORUNLU KONTROLLER:
1. npm run build (başarılı olmalı)
2. npm run dev (uygulama çalışmalı)
3. Manuel test (özellik çalışıyor mu?)
4. Regression test (mevcut özellikler bozulmadı mı?)
5. Performance check (yavaşlama var mı?)

MERGE ADIMI:
git checkout develop
git pull origin develop
git merge feature/[branch-name] --no-ff
git push origin develop
git branch -d feature/[branch-name]

CONFLICT RESOLUTION:
git checkout feature/[branch-name]
git rebase develop
# Conflict'leri manuel çöz
git add .
git rebase --continue
# Sonra normal merge yap
```

### **Rollback ve Emergency Protokolü**
```bash
SORUN DURUMUNDA:
1. Hemen durumu bildir
2. Son çalışan commit'i bul: git log --oneline
3. Rollback yap: git revert [commit-hash]
4. Hotfix branch aç: git checkout -b hotfix/emergency-fix
5. Sorunu çöz ve test et
6. Acil merge yap

CHECKPOINT SİSTEMİ:
Her major milestone sonrası:
git tag checkpoint-[phase]-[date]
git push origin --tags

EMERGENCY ROLLBACK:
git checkout main
git reset --hard [last-stable-commit]
git push --force-with-lease origin main
```

---

## 📊 **PROGRESS TRACKING ENTEGRASYONU**

### **Her Görev Öncesi**
```bash
1. PROGRESS_TRACKER.md dosyasını oku
2. Aktif görevi belirle (🔄 In Progress)
3. Görev detaylarını kontrol et
4. Bağımlılıkları kontrol et (dependent_task_ids)
5. Feature branch oluştur
```

### **Her Görev Sonrası**
```bash
1. PROGRESS_TRACKER.md'yi güncelle:
   - Tamamlanan görevi ✅ olarak işaretle
   - Commit hash'i ekle
   - Test sonuçlarını kaydet
   - Harcanan süreyi not et

2. Sonraki görevi belirle:
   - Bağımlılık kontrolü yap
   - Öncelik sırasını kontrol et
   - Kullanıcıya sonraki adımı sor
```

### **Günlük Rapor Formatı**
```markdown
## 📅 GÜNLÜK İLERLEME RAPORU - [TARIH]

### ✅ TAMAMLANAN GÖREVLER
- [x] 1.1.1 - PostgreSQL bağlantısı kuruldu
  - Branch: feature/1.1.1-database-connection
  - Commit: abc1234 - "feat: add PostgreSQL database connection"
  - Test: ✅ Build successful, connection working
  - Süre: 2 saat
  - Merge: ✅ develop branch'e merge edildi

### 🔄 DEVAM EDEN GÖREV
- [ ] 1.1.2 - API proxy entegrasyonu (50% tamamlandı)
  - Branch: feature/1.1.2-api-proxy-integration
  - Durum: Response format uyumsuzluğu çözülüyor
  - ETA: 1 saat

### ⏳ BEKLEYEN GÖREVLER
- [ ] 1.1.3 - Mock data'dan gerçek data'ya geçiş
- [ ] 1.2.1 - n8n webhook entegrasyonu

### 🚨 SORUNLAR
- Yok

### 📊 ÖZET
- Tamamlanan: 1/20 görev (%5)
- Harcanan süre: 2 saat
- Kalan tahmini süre: 38 saat
- Risk durumu: 🟢 Düşük
```

---

## 🚫 **YASAK İŞLEMLER VE KISITLAMALAR**

### **UI Değişiklik Kısıtlamaları**
```bash
YASAK İŞLEMLER:
❌ UI component yapısını değiştirme
❌ CSS class'larını değiştirme
❌ Tailwind konfigürasyonunu değiştirme
❌ Responsive breakpoint'leri değiştirme
❌ Color scheme'i değiştirme
❌ Layout yapısını değiştirme

UI DEĞİŞİKLİK PROTOKOLÜ:
1. "UI değişikliği gerekli" diye bildir
2. Kullanıcıdan onay bekle: "UI değişikliği onayla"
3. Onay gelirse minimal değişiklik yap
4. Değişiklik öncesi backup al
5. Test et ve onay iste

İZİNLİ İŞLEMLER:
✅ Data binding'i güncelleme
✅ API call'ları değiştirme
✅ State management'i iyileştirme
✅ Performance optimization
✅ Bug fix (UI'yı bozmadan)
```

### **Mock Data Yasağı**
```bash
KESİNLİKLE YASAK:
❌ const mockData = [...]
❌ const fakeVehicles = {...}
❌ Hard-coded arrays in components
❌ Static test data in code
❌ Dummy data objects

İZİNLİ VERİ KAYNAKLARI:
✅ PostgreSQL database queries
✅ n8n webhook data
✅ Proxy API responses
✅ Real-time API calls
✅ Database seed scripts (sadece DB'de)

DEVELOPMENT DATA:
- Test data sadece database'de
- SQL INSERT scripts ile
- Environment variable ile kontrol
- Production'da real data only
```

### **Kod Kalitesi Kuralları**
```bash
HER COMMIT ÖNCESİ:
- [ ] TypeScript errors yok
- [ ] ESLint warnings yok
- [ ] Build başarılı
- [ ] Console errors yok
- [ ] Performance regression yok
- [ ] Memory leaks yok

KOD STANDARTLARI:
- Google-style code formatting
- Meaningful variable names
- Comprehensive error handling
- Proper TypeScript types
- Clean component structure
- Optimized imports
```

---

## 🔧 **TEKNİK ENTEGRASYON KURALLARI**

### **Database İşlemleri**
```bash
DATABASE BAĞLANTISI:
- Connection pooling kullan
- Prepared statements kullan
- SQL injection koruması
- Transaction management
- Error handling ve retry logic

QUERY OPTİMİZASYONU:
- Index'leri kullan
- N+1 query problemini önle
- Pagination implement et
- Caching layer ekle
- Performance monitoring
```

### **API Entegrasyonu**
```bash
API CALLS:
- Timeout ayarları
- Retry mechanism
- Error handling
- Response validation
- Rate limiting respect
- Proper HTTP status codes

REAL-TIME FEATURES:
- WebSocket connections
- n8n webhook handling
- Event-driven updates
- State synchronization
- Connection recovery
```

### **Environment Management**
```bash
ENVIRONMENT VARIABLES:
- Sensitive data .env'de
- Production vs development configs
- API key rotation support
- Database connection strings
- Feature flags
- Debug mode controls
```

---

## 📝 **ÇALIŞMA AKIŞI ÖRNEĞİ**

### **Tipik Bir Görev Döngüsü**
```bash
# 1. Görev başlatma
git checkout develop
git pull origin develop
git checkout -b feature/1.2.1-n8n-webhook-integration

# 2. Geliştirme
# - Sadece n8n webhook ile ilgili kod yaz
# - Küçük commit'ler yap
git add .
git commit -m "feat: add n8n webhook endpoint configuration"

# 3. Test
npm run build  # ✅ Build successful
npm run dev    # ✅ App running
# Manuel test: Webhook çalışıyor mu?

# 4. Final commit
git add .
git commit -m "feat: complete n8n webhook integration

- Add webhook endpoint for real-time updates
- Implement data sync mechanism  
- Add error handling and retry logic
- Update UI to show live notifications

Tested: ✅ Build successful, webhooks working
Closes: Task 1.2.1"

# 5. Merge
git checkout develop
git merge feature/1.2.1-n8n-webhook-integration --no-ff
git push origin develop
git branch -d feature/1.2.1-n8n-webhook-integration

# 6. Progress update
# PROGRESS_TRACKER.md'yi güncelle
# Sonraki görevi sor
```

### **Sorun Durumu Yönetimi**
```bash
# Sorun tespit edildi
echo "🚨 SORUN TESPİT EDİLDİ: API timeout hatası"

# Hemen rollback
git revert HEAD

# Hotfix branch
git checkout -b hotfix/api-timeout-fix

# Sorunu çöz
# ... kod düzeltmeleri ...

# Test et
npm run build && npm run dev

# Acil merge
git checkout develop
git merge hotfix/api-timeout-fix
git push origin develop
git branch -d hotfix/api-timeout-fix

# Rapor et
echo "✅ SORUN ÇÖZÜLDİ: API timeout düzeltildi"
```

---

## 🎯 **BAŞARI KRİTERLERİ**

### **Her Görev İçin**
- ✅ Build başarılı
- ✅ Manuel test geçti
- ✅ Performance regression yok
- ✅ UI bozulmadı
- ✅ Mevcut özellikler çalışıyor
- ✅ Progress tracker güncel

### **Her Merge İçin**
- ✅ Conflict yok
- ✅ Test suite geçti
- ✅ Code review tamamlandı
- ✅ Documentation güncel
- ✅ Rollback planı hazır

### **Milestone Kriterleri**
- ✅ Tüm görevler tamamlandı
- ✅ Integration testleri geçti
- ✅ Performance hedefleri karşılandı
- ✅ User acceptance testleri geçti
- ✅ Production deployment hazır

---

**📅 Doküman Tarihi:** 2025-01-27  
**🎯 Amaç:** Git-based kontrollü geliştirme  
**⚡ Kural:** Her adım test edilebilir ve geri alınabilir  
**🔄 Güncelleme:** Her proje başında bu dosyayı oku  
**📊 Takip:** Progress tracker ile entegre çalış