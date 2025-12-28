# İş Paketi 2: Yapay Zeka, Karakter ve Bot Geliştirme

## Proje Bilgileri

| Bilgi | Değer |
|-------|-------|
| **Proje Adı** | Yapay Zeka Destekli Çok Oyunculu Coop (İşbirlikçi) TPS Oyununun Özelleştirilmiş Unreal Engine Oyun Motorunda Prototipinin Geliştirilmesi |
| **Kuruluş** | MB Oyun Yazılım ve Pazarlama A.Ş. |
| **Destek Programı** | TÜBİTAK 1501 - Sanayi Ar-Ge Projeleri Destekleme Programı |
| **İş Paketi** | İP-2: Yapay Zeka, Karakter ve Bot Geliştirme |
| **Süre** | 90 gün |
| **Bilimsel Danışman** | Prof. Dr. Turgay Bilgin (Bursa Teknik Üniversitesi) |

---

## İş Paketinin Amacı

Bu iş paketi, oyun içindeki karakter yapılarının, yapay zeka tabanlı bot sistemlerinin ve animasyon ağlarının geliştirilmesini kapsayan aşamaları içermektedir. 

**Temel Hedefler:**
- Karakter animasyonlarının senkronize çalışması
- Yapay zekanın oyuncu davranışlarına dinamik olarak tepki vermesi
- Botların oyun mekanikleri ile uyumlu hareket etmesi
- İleri düzey yapay zeka algoritmaları ve animasyon sistemleri kullanımı

---

## Kullanılacak Yöntemler ve Özgün Katkılar

### 1. Dürtü Tabanlı Yapay Zeka (Utility AI) Yaklaşımı

Botların statik davranış sergilemesini engellemek ve oyuncu etkileşimine göre dinamik kararlar almasını sağlamak amacıyla Utility AI yöntemi kullanılacaktır. Bu yöntem, botların ortam koşullarına, oyuncu pozisyonlarına ve tehdit analizlerine göre karar mekanizmasını şekillendirmesine olanak tanıyacaktır.

**Özgün Katkı:** Consideration sistemi ile çoklu faktörlerin eş zamanlı değerlendirilmesi ve response curve'ler ile dinamik skor hesaplama.

### 2. Davranış Ağacı (Behavior Tree) Kullanımı

Yapay zeka karakterlerinin belirli görevleri yerine getirebilmesi için hiyerarşik karar mekanizmaları içeren bir davranış ağacı modeli uygulanacaktır. Bu sayede botlar devriye gezme, saldırı yapma, kaçış stratejisi belirleme gibi eylemleri dinamik olarak gerçekleştirebilecektir.

**Özgün Katkı:** Custom Task ve Decorator node'ları ile oyuna özel davranış modülleri geliştirme.

### 3. Güçlendirmeli Öğrenme (Reinforcement Learning) ile Oyun Anlamlandırma

Oyuncuların oynayış tarzlarını anlamak ve botların buna uyum sağlamasını desteklemek amacıyla, takviyeli öğrenme teknikleri ile AI eğitimi planlanmaktadır. Bu süreç, botların savaş, kaçış, saklanma gibi stratejik kararlar almasına yardımcı olacaktır.

**Özgün Katkı:** PPO algoritması ile self-play eğitimi ve Python-Unreal Engine bridge ile gerçek zamanlı inference.

### 4. Gerçekçi Karakter Animasyonları İçin İleri Seviye Inverse Kinematics (IK) Kullanımı

Karakter animasyonlarının, zemin eğimine, düşman pozisyonlarına ve hareket hızına göre gerçekçi bir şekilde çalışması sağlanacaktır. Procedural Animation teknikleri ile karakterlerin otomatik animasyon uyarlamaları geliştirilmesi planlanmaktadır.

**Özgün Katkı:** Foot IK, Aim IK ve Look-At IK sistemlerinin Control Rig ile entegrasyonu.

### 5. Zorluk Seviyesine Göre Adaptif Düşman AI Modeli

Oyundaki botların zorluk seviyesini oyuncunun performansına göre dinamik olarak ayarlayan bir adaptif yapay zeka modeli tasarlanacaktır. Bu model, oyuncunun doğruluk oranı, tepki süresi, ortalama hayatta kalma süresi gibi parametreleri analiz ederek, botların zorluk seviyesini anlık olarak değiştirecektir.

**Özgün Katkı:** Player Performance Tracker ile metrik toplama ve Difficulty Manager ile yumuşak geçişli zorluk ayarlama.

---

## Bu Repodaki Dokümanlar

Bu depo, İş Paketi 2 kapsamında geliştirilen yapay zeka sistemlerinin teknik dokümantasyonunu içermektedir.

### 📄 Doküman Listesi

| Dosya Adı | Açıklama | İçerik |
|-----------|----------|--------|
| `ai-bot-mvp-teknik-rapor.md` | MVP Bot Yazılımı Teknik Raporu | Tüm AI sistemlerinin kod örnekleriyle birlikte adım adım implementasyon rehberi. C++ sınıfları, Blueprint yapıları, Python RL server kodu içerir. |
| `demo-ortami-tasarim-rehberi.md` | Hakem Demo Ortamı Tasarım Rehberi | TÜBİTAK hakem ziyareti için demo ortamının kurulumu, UI tasarımı, senaryo akışları ve sunum rehberi. |
| `oyun-yapay-zekasi-literatur-taramasi-tr.md` | Akademik Literatür Taraması | 2020-2025 yılları arasında yayınlanmış 18 hakemli makalenin APA formatında alıntıları ve özetleri. |

### 📁 Dosya Yapısı

```
📦 İş Paketi 2 - AI Bot Geliştirme
│
├── 📄 README.md (Bu dosya)
│
├── 📂 Teknik Dokümanlar
│   ├── Technical_Report.md
│   │   ├── Faz 1: AI Controller ve Temel Davranışlar
│   │   ├── Faz 2: Behavior Tree Sistemi
│   │   ├── Faz 3: Utility AI Entegrasyonu
│   │   ├── Faz 4: Güçlendirmeli Öğrenme (RL) Prototipi
│   │   ├── Faz 5: Adaptif Zorluk Sistemi
│   │   ├── Faz 6: Inverse Kinematics (IK) Sistemi
│   │   └── Faz 7: Test ve Debug Araçları
│   │
│   └── IzleyiciHakem_icin_demo.md
│       ├── Level Tasarımı
│       ├── Demo Kontrol Sistemi
│       ├── Debug UI Sistemi
│       ├── Bot Spawner Sistemi
│       ├── Kamera Sistemi
│       ├── Demo Senaryoları
│       └── Kurulum Adımları
│
└── 📂 Araştırma
    └── readme.md
        ├── Utility AI Makaleleri (3 adet)
        ├── Davranış Ağaçları Makaleleri (3 adet)
        ├── Güçlendirmeli Öğrenme Makaleleri (3 adet)
        ├── Dinamik Zorluk Ayarlama Makaleleri (3 adet)
        ├── Ters Kinematik Makaleleri (3 adet)
        └── NPC AI Sistemleri Makaleleri (3 adet)
```

---

## Teknik Mimari Özeti

### Kullanılan Teknolojiler

| Kategori | Teknoloji |
|----------|-----------|
| Oyun Motoru | Unreal Engine 5.3+ |
| Programlama Dilleri | C++, Blueprints, Python |
| AI Sistemleri | Behavior Trees, Utility AI, EQS |
| Makine Öğrenmesi | PyTorch, Stable-Baselines3, PPO |
| Haberleşme | gRPC, TCP/IP |
| Versiyon Kontrol | Git (Bitbucket) |

### Sistem Mimarisi

```
┌─────────────────────────────────────────────────────────────┐
│                    UNREAL ENGINE 5                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Bot         │  │ Behavior    │  │ Utility AI  │         │
│  │ Character   │──│ Tree        │──│ Component   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│         │                │                │                 │
│         ▼                ▼                ▼                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ AI          │  │ Perception  │  │ Navigation  │         │
│  │ Controller  │──│ System      │──│ System      │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────┐           │
│  │           Difficulty Manager                 │           │
│  │  ┌─────────────┐  ┌─────────────┐           │           │
│  │  │ Player      │  │ Bot Params  │           │           │
│  │  │ Performance │──│ Adjuster    │           │           │
│  │  └─────────────┘  └─────────────┘           │           │
│  └─────────────────────────────────────────────┘           │
│         │                                                   │
│         ▼ (gRPC)                                           │
├─────────────────────────────────────────────────────────────┤
│                    PYTHON RL SERVER                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ PPO Agent   │  │ Reward      │  │ Training    │         │
│  │             │──│ Calculator  │──│ Loop        │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## Zorluk Seviyeleri

Projede tanımlanan 7 farklı bot zorluk seviyesi:

| Seviye | Türkçe Adı | Sağlık | İsabet | Tepki Süresi | Agresiflik |
|--------|------------|--------|--------|--------------|------------|
| 1 | Yolcu | 50 | %20 | 1.0s | %10 |
| 2 | Mürettebat | 75 | %40 | 0.7s | %30 |
| 3 | Gemi Muhafızı | 100 | %60 | 0.5s | %50 |
| 4 | Gemi Polisi | 120 | %70 | 0.4s | %60 |
| 5 | Koruma | 150 | %80 | 0.3s | %70 |
| 6 | Sahil Güvenlik | 175 | %85 | 0.25s | %80 |
| 7 | Özel Kuvvetler | 200 | %95 | 0.15s | %90 |

---

## Hakem Demosu

TÜBİTAK hakem ziyaretinde gösterilecek 3 ana demo senaryosu:

### Senaryo 1: Temel Savaş Davranışları (5-7 dk)
- Bot spawn ve algılama sistemi
- Utility AI karar verme süreci
- Siper alma ve kaçış davranışları

### Senaryo 2: Adaptif Zorluk Sistemi (7-10 dk)
- Oyuncu performans metrikleri
- Dinamik zorluk ayarlama
- Bot parametre değişimleri

### Senaryo 3: RL Karşılaştırma (10-12 dk)
- Rule-based vs RL-trained bot
- Eğitim grafikleri (TensorBoard)
- Davranış farkları analizi

---

## Ar-Ge Niteliği ve Teknolojik Belirsizlikler

Bu iş paketi kapsamında ele alınan teknolojik belirsizlikler:

1. **Utility AI Skorlama:** Çoklu faktörlerin ağırlıklandırılması ve response curve tasarımı için optimal parametrelerin belirlenmesi
2. **RL Eğitim Stabilitesi:** Reward fonksiyonu tasarımı ve eğitim hiperparametrelerinin oyun ortamına uyarlanması
3. **Performans Optimizasyonu:** Çok sayıda botun eş zamanlı çalışmasında CPU/GPU kaynak yönetimi
4. **Adaptif Zorluk Dengesi:** Oyuncu deneyimini bozmadan zorluk geçişlerinin yumuşak yapılması

---

## İletişim

| Rol | İsim | Kurum |
|-----|------|-------|
| Proje Yürütücüsü | Erdoğan Atmaca | MB Oyun Yazılım ve Pazarlama A.Ş. |
| Bilimsel Danışman | Prof. Dr. Turgay Bilgin | Bursa Teknik Üniversitesi |

---

## Lisans

Bu dokümanlar TÜBİTAK 1501 Sanayi Ar-Ge Projeleri Destekleme Programı kapsamında hazırlanmıştır. Tüm hakları MB Oyun Yazılım ve Pazarlama A.Ş.'ye aittir.

---

*Son Güncelleme: 2025*
