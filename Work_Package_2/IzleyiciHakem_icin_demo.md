# Yapay Zeka Bot Demo Ortamı Tasarım Rehberi

**Proje:** Yapay Zeka Destekli Çok Oyunculu Coop TPS Oyunu  
**Doküman:** Hakem Demo Ortamı Kurulum Rehberi  
**Versiyon:** 1.0  
**Tarih:** 28.12.2025

---

## İçindekiler

1. [Demo Ortamı Genel Bakış](#1-demo-ortamı-genel-bakış)
2. [Level Tasarımı](#2-level-tasarımı)
3. [Demo Kontrol Sistemi](#3-demo-kontrol-sistemi)
4. [Debug UI Sistemi](#4-debug-ui-sistemi)
5. [Bot Spawner Sistemi](#5-bot-spawner-sistemi)
6. [Kamera Sistemi](#6-kamera-sistemi)
7. [Demo Senaryoları ve Akış](#7-demo-senaryoları-ve-akış)
8. [Klavye Kısayolları](#8-klavye-kısayolları)
9. [Kurulum Adımları](#9-kurulum-adımları)

---

## 1. Demo Ortamı Genel Bakış

### 1.1 Amaç

Bu demo ortamı, TÜBİTAK hakemlerine aşağıdaki sistemleri görsel ve interaktif olarak sunmak için tasarlanmıştır:

- Utility AI karar verme mekanizması
- Behavior Tree hiyerarşik davranış sistemi
- Adaptif zorluk ayarlama (DDA)
- Güçlendirmeli öğrenme (RL) prototipi
- Bot algılama ve navigasyon sistemleri

### 1.2 Demo Ortamı Bileşenleri

```
Demo Ortamı
├── Demo Level (L_AIDemo)
│   ├── Arena alanı
│   ├── Siper pozisyonları
│   ├── Spawn noktaları
│   └── Devriye rotaları
│
├── Kontrol Sistemi
│   ├── BP_DemoController (Ana kontrol)
│   ├── BP_BotSpawner (Bot oluşturma)
│   └── BP_ScenarioManager (Senaryo yönetimi)
│
├── UI Sistemi
│   ├── WBP_DemoHUD (Ana HUD)
│   ├── WBP_BotDebugPanel (Bot debug bilgileri)
│   ├── WBP_DifficultyMeter (Zorluk göstergesi)
│   ├── WBP_UtilityAIGraph (Utility AI skorları)
│   └── WBP_ControlPanel (Kontrol paneli)
│
└── Kamera Sistemi
    ├── Oyuncu kamerası
    ├── Serbest uçuş kamerası
    └── Bot takip kamerası
```

---

## 2. Level Tasarımı

### 2.1 Arena Düzeni

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    [Spawn_Player]                              [Camera_01]      │
│         ●                                          ◎            │
│                                                                 │
│    ╔═══════════╗                          ╔═══════════╗        │
│    ║  Cover_01 ║                          ║  Cover_02 ║        │
│    ╚═══════════╝                          ╚═══════════╝        │
│                                                                 │
│                      [Patrol_Route_01]                         │
│                    ●───────●───────●                           │
│                                                                 │
│         ╔═══════════╗              ╔═══════════╗               │
│         ║  Cover_03 ║              ║  Cover_04 ║               │
│         ╚═══════════╝              ╚═══════════╝               │
│                                                                 │
│                         [Central_Area]                          │
│                              ★                                  │
│                                                                 │
│         ╔═══════════╗              ╔═══════════╗               │
│         ║  Cover_05 ║              ║  Cover_06 ║               │
│         ╚═══════════╝              ╚═══════════╝               │
│                                                                 │
│                    ●───────●───────●                           │
│                      [Patrol_Route_02]                         │
│                                                                 │
│    ╔═══════════╗                          ╔═══════════╗        │
│    ║  Cover_07 ║                          ║  Cover_08 ║        │
│    ╚═══════════╝                          ╚═══════════╝        │
│                                                                 │
│    [Spawn_Bot_01]    [Spawn_Bot_02]    [Spawn_Bot_03]          │
│         ●                 ●                 ●                   │
│                                                                 │
│                                                [Camera_02]      │
│                                                    ◎            │
└─────────────────────────────────────────────────────────────────┘

Boyutlar: 50m x 50m (5000 x 5000 Unreal Units)
```

### 2.2 Level Blueprint Yapısı

```cpp
// L_AIDemo Level Blueprint - Event Graph

// ===== BEGIN PLAY =====
Event BeginPlay
    │
    ├──► Initialize Demo Controller
    │       └──► Spawn BP_DemoController at (0,0,0)
    │
    ├──► Setup UI
    │       ├──► Create WBP_DemoHUD widget
    │       └──► Add to Viewport
    │
    ├──► Setup Cameras
    │       ├──► Find all CameraActors with tag "DemoCamera"
    │       └──► Store in CameraArray
    │
    └──► Start Default Scenario
            └──► Call "StartScenario_BasicCombat"
```

### 2.3 Cover Actor Blueprint

```cpp
// BP_CoverActor.h - Siper noktası aktörü

UCLASS()
class ABP_CoverActor : public AActor
{
    GENERATED_BODY()

public:
    // Siper tipi
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cover")
    ECoverType CoverType = ECoverType::Half;  // Half, Full, Destructible

    // Siper yönü (düşmana karşı korunan yön)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cover")
    FVector CoverDirection;

    // Kapasite (kaç bot sığabilir)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cover")
    int32 MaxOccupants = 2;

    // Mevcut kullananlar
    UPROPERTY(BlueprintReadOnly, Category = "Cover")
    TArray<AActor*> CurrentOccupants;

    // Siper pozisyonları
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cover")
    TArray<FVector> CoverPositions;

    // EQS için sorgulanabilir
    UFUNCTION(BlueprintCallable, Category = "Cover")
    bool IsAvailable() const { return CurrentOccupants.Num() < MaxOccupants; }

    UFUNCTION(BlueprintCallable, Category = "Cover")
    FVector GetBestCoverPosition(AActor* Requester, FVector ThreatLocation);
};
```

**Blueprint Görsel Kurulum:**

```
BP_CoverActor Components:
├── DefaultSceneRoot
├── StaticMesh (Siper geometrisi)
│   └── SM_CoverWall_Half veya SM_CoverWall_Full
├── BoxCollision (EQS query için)
│   └── Collision Preset: OverlapAllDynamic
├── ArrowComponent (Siper yönünü gösterir)
└── WidgetComponent (Debug için - opsiyonel)
    └── WBP_CoverDebug
```

### 2.4 Patrol Route Actor

```cpp
// BP_PatrolRoute - Devriye rotası

UCLASS()
class ABP_PatrolRoute : public AActor
{
    GENERATED_BODY()

public:
    // Rota noktaları
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Patrol")
    TArray<FVector> PatrolPoints;

    // Döngüsel mü?
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Patrol")
    bool bIsLooping = true;

    // Her noktada bekleme süresi
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Patrol")
    float WaitTimeAtPoints = 2.0f;

    // Debug çizimi
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bDrawDebug = true;

    // Spline component ile görselleştirme
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
    USplineComponent* PatrolSpline;
};
```

---

## 3. Demo Kontrol Sistemi

### 3.1 Ana Demo Controller

```cpp
// BP_DemoController.h

UCLASS()
class ABP_DemoController : public AActor
{
    GENERATED_BODY()

public:
    ABP_DemoController();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // ===== SENARYO KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Scenarios")
    void StartScenario(EDemoScenario Scenario);

    UFUNCTION(BlueprintCallable, Category = "Demo|Scenarios")
    void StopCurrentScenario();

    UFUNCTION(BlueprintCallable, Category = "Demo|Scenarios")
    void ResetDemo();

    // ===== BOT KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Bots")
    void SpawnBot(EBotDifficultyLevel Difficulty, FVector Location);

    UFUNCTION(BlueprintCallable, Category = "Demo|Bots")
    void SpawnBotAtSpawnPoint(int32 SpawnPointIndex, EBotDifficultyLevel Difficulty);

    UFUNCTION(BlueprintCallable, Category = "Demo|Bots")
    void KillAllBots();

    UFUNCTION(BlueprintCallable, Category = "Demo|Bots")
    void SetAllBotsDifficulty(EBotDifficultyLevel NewDifficulty);

    UFUNCTION(BlueprintCallable, Category = "Demo|Bots")
    TArray<ABotCharacter*> GetAllBots() const { return ActiveBots; }

    // ===== ZORLUK KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Difficulty")
    void SetGlobalDifficulty(float Difficulty);

    UFUNCTION(BlueprintCallable, Category = "Demo|Difficulty")
    void ToggleAdaptiveDifficulty(bool bEnabled);

    UFUNCTION(BlueprintCallable, Category = "Demo|Difficulty")
    float GetCurrentDifficulty() const;

    // ===== DEBUG KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Debug")
    void ToggleDebugMode(bool bEnabled);

    UFUNCTION(BlueprintCallable, Category = "Demo|Debug")
    void SetDebugFlags(bool bBehaviorTree, bool bUtilityAI, 
        bool bPerception, bool bNavigation);

    UFUNCTION(BlueprintCallable, Category = "Demo|Debug")
    void FocusOnBot(ABotCharacter* Bot);

    // ===== KAMERA KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Camera")
    void SwitchCamera(int32 CameraIndex);

    UFUNCTION(BlueprintCallable, Category = "Demo|Camera")
    void EnableFreeCamera(bool bEnabled);

    UFUNCTION(BlueprintCallable, Category = "Demo|Camera")
    void FollowBot(ABotCharacter* Bot);

    // ===== ZAMAN KONTROL =====
    UFUNCTION(BlueprintCallable, Category = "Demo|Time")
    void SetTimeScale(float Scale);

    UFUNCTION(BlueprintCallable, Category = "Demo|Time")
    void PauseDemo();

    UFUNCTION(BlueprintCallable, Category = "Demo|Time")
    void ResumeDemo();

protected:
    // Referanslar
    UPROPERTY()
    TArray<ABotCharacter*> ActiveBots;

    UPROPERTY()
    TArray<AActor*> SpawnPoints;

    UPROPERTY()
    TArray<ACameraActor*> DemoCameras;

    UPROPERTY()
    class UDifficultyManager* DifficultyManager;

    // Mevcut durum
    UPROPERTY()
    EDemoScenario CurrentScenario;

    UPROPERTY()
    bool bIsDebugMode = true;

    UPROPERTY()
    float CurrentTimeScale = 1.0f;

private:
    void FindAllSpawnPoints();
    void FindAllCameras();
    void SetupInputBindings();
};
```

### 3.2 Demo Controller Blueprint Event Graph

```
// BP_DemoController - Event Graph

// ===== INPUT BINDINGS =====

InputAction "ToggleDebug" (F1)
    └──► ToggleDebugMode(!bIsDebugMode)

InputAction "SpawnBot" (F2)
    └──► SpawnBotAtSpawnPoint(0, CurrentDifficulty)

InputAction "KillBots" (F3)
    └──► KillAllBots()

InputAction "ResetDemo" (F4)
    └──► ResetDemo()

InputAction "CycleCamera" (F5)
    └──► SwitchCamera((CurrentCameraIndex + 1) % CameraCount)

InputAction "TogglePause" (P)
    └──► Branch: IsPaused?
            ├──► True: ResumeDemo()
            └──► False: PauseDemo()

InputAction "IncDifficulty" (NumPad+)
    └──► SetGlobalDifficulty(CurrentDifficulty + 0.1)

InputAction "DecDifficulty" (NumPad-)
    └──► SetGlobalDifficulty(CurrentDifficulty - 0.1)

InputAction "Scenario1" (1)
    └──► StartScenario(BasicCombat)

InputAction "Scenario2" (2)
    └──► StartScenario(AdaptiveDifficulty)

InputAction "Scenario3" (3)
    └──► StartScenario(RLComparison)

InputAction "TimeScaleUp" (])
    └──► SetTimeScale(CurrentTimeScale * 1.5)

InputAction "TimeScaleDown" ([)
    └──► SetTimeScale(CurrentTimeScale / 1.5)

InputAction "TimeScaleReset" (\)
    └──► SetTimeScale(1.0)
```

### 3.3 Senaryo Tanımları

```cpp
// DemoScenarios.h

UENUM(BlueprintType)
enum class EDemoScenario : uint8
{
    None,
    BasicCombat,           // Temel savaş davranışları
    AdaptiveDifficulty,    // DDA gösterimi
    RLComparison,          // RL vs Rule-based karşılaştırma
    TeamCoordination,      // Takım koordinasyonu
    PerceptionDemo,        // Algılama sistemleri
    NavigationDemo,        // Pathfinding ve navigasyon
    UtilityAIShowcase,     // Utility AI detaylı gösterim
    StressTest             // Çok sayıda bot performans testi
};

USTRUCT(BlueprintType)
struct FDemoScenarioConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EDemoScenario ScenarioType;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText ScenarioName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText Description;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 BotCount = 3;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<EBotDifficultyLevel> BotDifficulties;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float InitialDifficulty = 0.5f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bAdaptiveDifficultyEnabled = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<int32> SpawnPointIndices;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bUseRLBots = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 CameraIndex = 0;
};
```

---

## 4. Debug UI Sistemi

### 4.1 Ana HUD Widget (WBP_DemoHUD)

```
WBP_DemoHUD Hierarchy:
├── CanvasPanel (Root)
│   │
│   ├── [TOP LEFT] - Senaryo Bilgisi
│   │   └── VerticalBox
│   │       ├── Text_ScenarioName
│   │       ├── Text_ScenarioDescription
│   │       └── Text_TimeScale
│   │
│   ├── [TOP CENTER] - Zorluk Göstergesi
│   │   └── WBP_DifficultyMeter
│   │
│   ├── [TOP RIGHT] - Kontrol Paneli
│   │   └── WBP_ControlPanel
│   │
│   ├── [LEFT SIDE] - Bot Listesi
│   │   └── ScrollBox_BotList
│   │       └── [Dinamik] WBP_BotListItem (her bot için)
│   │
│   ├── [RIGHT SIDE] - Seçili Bot Detayları
│   │   └── WBP_BotDebugPanel
│   │
│   ├── [BOTTOM CENTER] - Utility AI Grafik
│   │   └── WBP_UtilityAIGraph
│   │
│   └── [BOTTOM] - Kısayol Bilgisi
│       └── HorizontalBox_Shortcuts
│           ├── Text: "F1: Debug"
│           ├── Text: "F2: Spawn"
│           ├── Text: "F3: Kill All"
│           ├── Text: "1-3: Senaryolar"
│           └── Text: "P: Pause"
```

### 4.2 Zorluk Göstergesi Widget (WBP_DifficultyMeter)

```cpp
// WBP_DifficultyMeter.cpp

UCLASS()
class UWBPDifficultyMeter : public UUserWidget
{
    GENERATED_BODY()

public:
    // UI Bileşenleri
    UPROPERTY(meta = (BindWidget))
    UProgressBar* ProgressBar_Difficulty;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* Text_DifficultyValue;

    UPROPERTY(meta = (BindWidget))
    UTextBlock* Text_DifficultyLabel;

    UPROPERTY(meta = (BindWidget))
    UImage* Image_AdaptiveIndicator;

    // Güncelleme
    UFUNCTION(BlueprintCallable, Category = "Difficulty")
    void UpdateDifficulty(float NewDifficulty, bool bIsAdaptive);

    UFUNCTION(BlueprintCallable, Category = "Difficulty")
    void SetTargetDifficulty(float Target);

protected:
    virtual void NativeTick(const FGeometry& MyGeometry, float DeltaTime) override;

private:
    float CurrentDisplayDifficulty = 0.5f;
    float TargetDifficulty = 0.5f;
    float InterpSpeed = 3.0f;

    FLinearColor GetDifficultyColor(float Difficulty);
};
```

**Blueprint Görsel Tasarım:**

```
┌─────────────────────────────────────┐
│         ZORLUK SEVİYESİ             │
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐    │
│  │█████████████░░░░░░░░░░░░░░░│    │  ← Progress Bar
│  └─────────────────────────────┘    │
│         52% (ORTA)                  │
│      [●] Adaptif Aktif              │  ← Yeşil nokta yanıp söner
└─────────────────────────────────────┘
```

### 4.3 Bot Debug Panel (WBP_BotDebugPanel)

```
WBP_BotDebugPanel Layout:
┌─────────────────────────────────────────┐
│  BOT: ShipGuard_03                      │
│  ════════════════════════════════════   │
│                                         │
│  DURUM                                  │
│  ├── Sağlık: ████████░░ 80%            │
│  ├── Mermi:  ██████████ 100%           │
│  └── Hız:    450 u/s                   │
│                                         │
│  ALGILAMA                               │
│  ├── Hedef: Player_01                   │
│  ├── Mesafe: 1250 units                 │
│  ├── Görüş: ✓ Var                       │
│  └── Tehdit: ████░░░░░░ 40%            │
│                                         │
│  KARAR (Utility AI)                     │
│  ├── Attack:     0.72 ████████░░       │
│  ├── TakeCover:  0.45 █████░░░░░       │
│  ├── Flee:       0.12 ██░░░░░░░░       │
│  ├── Reload:     0.08 █░░░░░░░░░       │
│  └── AKTIF: Attack ◄────────────       │
│                                         │
│  BEHAVIOR TREE                          │
│  └── Combat > Attack > FireWeapon      │
│                                         │
│  PARAMETRELERİ                          │
│  ├── Accuracy:   70%                    │
│  ├── Reaction:   0.30s                  │
│  ├── Aggression: 60%                    │
│  └── Flank Prob: 30%                    │
└─────────────────────────────────────────┘
```

### 4.4 Utility AI Graph Widget (WBP_UtilityAIGraph)

```cpp
// WBP_UtilityAIGraph - Çubuk grafik gösterimi

UCLASS()
class UWBPUtilityAIGraph : public UUserWidget
{
    GENERATED_BODY()

public:
    UPROPERTY(meta = (BindWidget))
    UHorizontalBox* HBox_Bars;

    UPROPERTY(EditAnywhere, Category = "Graph")
    TMap<EBotAction, FLinearColor> ActionColors;

    UFUNCTION(BlueprintCallable)
    void UpdateScores(const TMap<EBotAction, float>& Scores, EBotAction CurrentAction);

    UFUNCTION(BlueprintCallable)
    void SetTargetBot(ABotCharacter* Bot);

protected:
    virtual void NativeConstruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float DeltaTime) override;

private:
    UPROPERTY()
    TMap<EBotAction, UProgressBar*> ActionBars;

    UPROPERTY()
    TMap<EBotAction, UTextBlock*> ActionLabels;

    UPROPERTY()
    ABotCharacter* TargetBot;

    void CreateBarForAction(EBotAction Action);
    void AnimateBar(UProgressBar* Bar, float TargetValue, float DeltaTime);
};
```

**Görsel Tasarım:**

```
┌──────────────────────────────────────────────────────────────────┐
│                      UTILITY AI SKORLARI                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Attack     ████████████████████████░░░░░░  0.72  ◄── AKTIF     │
│  TakeCover  █████████████░░░░░░░░░░░░░░░░░  0.45                │
│  Flee       ████░░░░░░░░░░░░░░░░░░░░░░░░░░  0.12                │
│  Reload     ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.08                │
│  Patrol     █░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.03                │
│  Flank      ████████░░░░░░░░░░░░░░░░░░░░░░  0.28                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Renk Kodları:
- Attack:     Kırmızı (#FF4444)
- TakeCover:  Mavi (#4444FF)
- Flee:       Sarı (#FFFF44)
- Reload:     Turuncu (#FF8844)
- Patrol:     Yeşil (#44FF44)
- Flank:      Mor (#FF44FF)
```

### 4.5 Kontrol Paneli Widget (WBP_ControlPanel)

```
WBP_ControlPanel Layout:
┌─────────────────────────────────┐
│      DEMO KONTROL PANELİ        │
├─────────────────────────────────┤
│                                 │
│  SENARYO                        │
│  ┌─────────────────────────┐    │
│  │ ▼ Temel Savaş Demo     │    │  ← Dropdown
│  └─────────────────────────┘    │
│  [Başlat] [Durdur] [Sıfırla]   │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  BOT KONTROL                    │
│  Zorluk: ┌──────────────────┐   │
│          │ ▼ Gemi Muhafızı  │   │  ← Dropdown
│          └──────────────────┘   │
│  [+Bot Ekle]  [Tümünü Öldür]   │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  ZORLUK AYARI                   │
│  ◄─────────●─────────►         │  ← Slider
│        0.52                     │
│  [x] Adaptif Zorluk Aktif      │  ← Checkbox
│                                 │
│  ─────────────────────────────  │
│                                 │
│  DEBUG GÖRÜNÜM                  │
│  [x] Behavior Tree              │
│  [x] Utility AI                 │
│  [x] Perception                 │
│  [ ] Navigation                 │
│  [ ] IK Bones                   │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  ZAMAN                          │
│  ◄─────●───────────────►       │  ← Slider (0.1x - 3x)
│      0.5x                       │
│  [▶ Devam] [⏸ Duraklat]        │
│                                 │
└─────────────────────────────────┘
```

---

## 5. Bot Spawner Sistemi

### 5.1 Spawn Point Actor

```cpp
// BP_BotSpawnPoint.h

UCLASS()
class ABP_BotSpawnPoint : public AActor
{
    GENERATED_BODY()

public:
    ABP_BotSpawnPoint();

    // Spawn ayarları
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawn")
    EBotDifficultyLevel DefaultDifficulty = EBotDifficultyLevel::Crew;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawn")
    TSubclassOf<ABotCharacter> BotClass;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawn")
    int32 SpawnPointIndex = 0;

    // Atanan devriye rotası
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Spawn")
    ABP_PatrolRoute* AssignedPatrolRoute;

    // Spawn fonksiyonu
    UFUNCTION(BlueprintCallable, Category = "Spawn")
    ABotCharacter* SpawnBot(EBotDifficultyLevel OverrideDifficulty = EBotDifficultyLevel::Crew);

    // Debug
    UPROPERTY(EditAnywhere, Category = "Debug")
    bool bShowDebugSphere = true;

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    USphereComponent* SpawnRadius;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    UArrowComponent* SpawnDirection;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    UBillboardComponent* EditorIcon;
};
```

### 5.2 Bot Spawner Manager

```cpp
// BP_BotSpawnerManager.h

UCLASS()
class ABP_BotSpawnerManager : public AActor
{
    GENERATED_BODY()

public:
    // Tüm spawn point'leri bul
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void CollectSpawnPoints();

    // Belirli noktada spawn
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    ABotCharacter* SpawnBotAt(int32 SpawnPointIndex, 
        EBotDifficultyLevel Difficulty, bool bUseRL = false);

    // Rastgele noktada spawn
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    ABotCharacter* SpawnBotRandom(EBotDifficultyLevel Difficulty);

    // Çoklu spawn
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    TArray<ABotCharacter*> SpawnMultipleBots(int32 Count, 
        EBotDifficultyLevel Difficulty, bool bRandomPositions = true);

    // Wave spawn
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void StartWave(const FWaveConfig& WaveConfig);

    // Tüm botları öldür
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    void KillAllBots();

    // Aktif bot sayısı
    UFUNCTION(BlueprintCallable, Category = "Spawner")
    int32 GetActiveBotCount() const { return ActiveBots.Num(); }

    // Botlar
    UPROPERTY(BlueprintReadOnly, Category = "Spawner")
    TArray<ABotCharacter*> ActiveBots;

protected:
    UPROPERTY()
    TArray<ABP_BotSpawnPoint*> SpawnPoints;

    // Bot ölümünde çağrılır
    UFUNCTION()
    void OnBotDeath(ABotCharacter* DeadBot);
};
```

---

## 6. Kamera Sistemi

### 6.1 Demo Kamera Controller

```cpp
// BP_DemoCameraController.h

UENUM(BlueprintType)
enum class EDemoCameraMode : uint8
{
    Fixed,          // Sabit kamera pozisyonları
    FreeRoam,       // Serbest uçuş
    FollowBot,      // Bot takip
    FollowPlayer,   // Oyuncu takip
    Overview,       // Kuş bakışı
    Cinematic       // Sinematik geçişler
};

UCLASS()
class ABP_DemoCameraController : public AActor
{
    GENERATED_BODY()

public:
    ABP_DemoCameraController();

    // Kamera modu
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void SetCameraMode(EDemoCameraMode NewMode);

    // Sabit kameralar arası geçiş
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void SwitchToCamera(int32 CameraIndex, float BlendTime = 0.5f);

    // Bot takip
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void FollowBot(ABotCharacter* Bot);

    // Serbest kamera kontrolü
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void EnableFreeCamera(bool bEnabled);

    // Kuş bakışı
    UFUNCTION(BlueprintCallable, Category = "Camera")
    void SetOverviewCamera();

protected:
    UPROPERTY(EditAnywhere, Category = "Camera")
    TArray<ACameraActor*> FixedCameras;

    UPROPERTY()
    EDemoCameraMode CurrentMode;

    UPROPERTY()
    ABotCharacter* FollowTarget;

    UPROPERTY()
    ACameraActor* ActiveCamera;

    // Serbest kamera için
    UPROPERTY(EditAnywhere, Category = "Camera|FreeRoam")
    float FreeRoamSpeed = 1000.0f;

    UPROPERTY(EditAnywhere, Category = "Camera|FreeRoam")
    float FreeRoamRotationSpeed = 100.0f;

    // Takip kamera ayarları
    UPROPERTY(EditAnywhere, Category = "Camera|Follow")
    FVector FollowOffset = FVector(-300, 0, 200);

    UPROPERTY(EditAnywhere, Category = "Camera|Follow")
    float FollowInterpSpeed = 5.0f;

    virtual void Tick(float DeltaTime) override;

private:
    void UpdateFollowCamera(float DeltaTime);
    void UpdateFreeCamera(float DeltaTime);
};
```

### 6.2 Kamera Pozisyonları

```
Level'daki Kamera Pozisyonları:

Camera_01 (Overview - Kuş Bakışı)
├── Pozisyon: (0, 0, 3000)
├── Rotasyon: (-90, 0, 0) - Aşağı bakıyor
└── FOV: 90

Camera_02 (Side View - Yan Görünüm)
├── Pozisyon: (3000, 0, 500)
├── Rotasyon: (0, 180, 0) - Arena'ya bakıyor
└── FOV: 75

Camera_03 (Player Spawn View)
├── Pozisyon: (-2000, 500, 300)
├── Rotasyon: (0, 45, 0)
└── FOV: 70

Camera_04 (Bot Spawn View)
├── Pozisyon: (2000, -500, 300)
├── Rotasyon: (0, -135, 0)
└── FOV: 70

Camera_05 (Close Combat View)
├── Pozisyon: (0, 1500, 200)
├── Rotasyon: (-10, -90, 0)
└── FOV: 60
```

---

## 7. Demo Senaryoları ve Akış

### 7.1 Senaryo 1: Temel Savaş Davranışları

**Amaç:** Botların temel AI davranışlarını göstermek

**Süre:** 5-7 dakika

**Akış:**

```
1. BAŞLANGIÇ (0:00)
   ├── Sahne: Boş arena, oyuncu spawn noktasında
   ├── UI: Debug paneli aktif, Utility AI grafiği görünür
   └── Anlatım: "Şimdi botların temel karar verme mekanizmasını göreceğiz"

2. İLK BOT SPAWN (0:30)
   ├── Aksiyon: F2 ile tek bot spawn
   ├── Bot: ShipGuard seviyesi
   ├── Görsel: Bot devriye modunda başlıyor
   └── UI: Utility AI'da Patrol skoru yüksek

3. ALGILAMA DEMOsu (1:00)
   ├── Aksiyon: Oyuncu bota yaklaşıyor
   ├── Görsel: Debug'da perception range görünür
   ├── Tetikleme: Bot oyuncuyu algılıyor
   └── UI: IsAlert = TRUE, Attack skoru yükseliyor

4. SALDIRI DAVRANIŞI (1:30)
   ├── Durum: Bot saldırı moduna geçiyor
   ├── Görsel: Behavior Tree'de Combat > Attack aktif
   ├── UI: Attack skoru en yüksek (0.8+)
   └── Anlatım: "Bot şu an saldırı aksiyonunu tercih ediyor"

5. HASAR VE SİPER (2:30)
   ├── Aksiyon: Oyuncu bota hasar veriyor
   ├── Tepki: Bot'un canı düşüyor
   ├── UI: TakeCover skoru yükseliyor
   └── Görsel: Bot en yakın sipere koşuyor

6. KAÇIŞ DAVRANIŞI (3:30)
   ├── Durum: Bot'un canı kritik seviyede (%20 altı)
   ├── UI: Flee skoru dominant
   ├── Görsel: Bot oyuncudan uzaklaşıyor
   └── Anlatım: "Düşük sağlıkta kaçış davranışı aktif"

7. ÇOKLU BOT (4:30)
   ├── Aksiyon: 2 bot daha spawn
   ├── Görsel: Botlar arası koordinasyon
   ├── UI: Her botun kendi Utility AI skorları
   └── Anlatım: "Birden fazla bot farklı kararlar verebilir"

8. BİTİŞ (6:00)
   └── Özet ve soru-cevap
```

### 7.2 Senaryo 2: Adaptif Zorluk Sistemi

**Amaç:** DDA'nın nasıl çalıştığını göstermek

**Süre:** 7-10 dakika

**Akış:**

```
1. BAŞLANGIÇ (0:00)
   ├── UI: Zorluk göstergesi ekranın üstünde
   ├── Başlangıç zorluğu: %50
   └── Adaptif mod: AÇIK

2. DÜŞÜK PERFORMANS SİMÜLASYONU (1:00)
   ├── Aksiyon: Oyuncu kasıtlı olarak kötü oynuyor
   ├── Metrikler: K/D düşük, ölümler artıyor
   ├── Sonuç: Zorluk yavaşça düşüyor
   └── UI: %50 → %40 → %35...

3. ZORLUK DEĞİŞİMİNİ GÖZLEMLEME (2:00)
   ├── Bot parametreleri gösteriliyor:
   │   ├── Accuracy: %70 → %50
   │   ├── Reaction: 0.3s → 0.5s
   │   └── Aggression: %60 → %40
   └── Anlatım: "Sistem oyuncuya uyum sağlıyor"

4. YÜKSEK PERFORMANS (4:00)
   ├── Aksiyon: Oyuncu çok iyi oynuyor
   ├── Metrikler: K/D yükseliyor
   ├── Sonuç: Zorluk artıyor
   └── UI: %35 → %50 → %65...

5. MANUEL KONTROL (6:00)
   ├── Adaptif modu KAPAT
   ├── Slider ile manuel zorluk ayarla
   ├── Anlık değişimleri göster
   └── Anlatım: "Manuel kontrol de mümkün"

6. KARŞILAŞTIRMA (8:00)
   ├── İki bot yan yana:
   │   ├── Bot A: Düşük zorluk (%20)
   │   └── Bot B: Yüksek zorluk (%80)
   ├── Aynı anda saldırıyorlar
   └── Fark açıkça görülüyor
```

### 7.3 Senaryo 3: RL Karşılaştırma

**Amaç:** RL eğitimli bot ile rule-based bot farkını göstermek

**Süre:** 10-12 dakika

**Akış:**

```
1. GİRİŞ (0:00)
   ├── Açıklama: İki farklı AI yaklaşımı
   ├── Sol: Behavior Tree (Rule-based)
   └── Sağ: Reinforcement Learning

2. AYNI SENARYO, İKİ BOT (1:00)
   ├── Spawn: İki bot aynı pozisyonda
   ├── Renk kodları:
   │   ├── Mavi: BT Bot
   │   └── Turuncu: RL Bot
   └── UI: Her ikisinin de aksiyonları görünür

3. DAVRANIŞ FARKLARI (3:00)
   ├── Durum: Aynı tehdit
   ├── BT Bot: Önceden tanımlı kurallara göre
   └── RL Bot: Öğrenilmiş davranışlar

4. TENSORBOARD GÖSTERİMİ (5:00)
   ├── Eğitim grafikleri
   ├── Reward eğrisi
   ├── Episode uzunlukları
   └── Anlatım: "Bu bot X saat eğitildi"

5. EDGE CASE'LER (7:00)
   ├── Beklenmedik durumlar oluştur
   ├── BT: Tanımlı değilse takılabilir
   └── RL: Adaptif davranış

6. SONUÇ VE TARTIŞMA (10:00)
   ├── Her iki yaklaşımın avantajları
   ├── Hibrit sistem önerisi
   └── Soru-cevap
```

---

## 8. Klavye Kısayolları

### 8.1 Demo Kontrol Kısayolları

| Tuş | Fonksiyon | Açıklama |
|-----|-----------|----------|
| F1 | Toggle Debug | Debug görünümünü aç/kapa |
| F2 | Spawn Bot | Yeni bot spawn et |
| F3 | Kill All | Tüm botları öldür |
| F4 | Reset Demo | Demoyu sıfırla |
| F5 | Cycle Camera | Kameralar arası geçiş |
| F6 | Free Camera | Serbest kamera modu |
| F7 | Follow Bot | Seçili botu takip et |
| F8 | Screenshot | Ekran görüntüsü al |

### 8.2 Senaryo Kısayolları

| Tuş | Senaryo |
|-----|---------|
| 1 | Temel Savaş Demo |
| 2 | Adaptif Zorluk Demo |
| 3 | RL Karşılaştırma |
| 4 | Takım Koordinasyonu |
| 5 | Perception Demo |

### 8.3 Zaman ve Zorluk Kısayolları

| Tuş | Fonksiyon |
|-----|-----------|
| P | Pause/Resume |
| [ | Zaman yavaşlat (0.5x) |
| ] | Zaman hızlandır (2x) |
| \ | Normal hız (1x) |
| NumPad + | Zorluk artır (+0.1) |
| NumPad - | Zorluk azalt (-0.1) |
| NumPad * | Adaptif toggle |

### 8.4 Bot Seçim Kısayolları

| Tuş | Fonksiyon |
|-----|-----------|
| Tab | Sonraki botu seç |
| Shift+Tab | Önceki botu seç |
| Click on Bot | O botu seç |
| Escape | Seçimi kaldır |

---

## 9. Kurulum Adımları

### 9.1 Adım Adım Kurulum

```
ADIM 1: Level Oluşturma
────────────────────────
1. File > New Level > Empty Level
2. Kaydet: Content/Maps/L_AIDemo
3. World Settings:
   - Game Mode: BP_DemoGameMode (oluşturulacak)
   - NavMesh Bounds Volume ekle (tüm arena'yı kapla)

ADIM 2: Arena Geometrisi
────────────────────────
1. Zemin: Plane mesh (5000x5000)
2. Duvarlar: 4 adet Box collision (görünmez)
3. Siperler: BP_CoverActor yerleştir (8 adet)
4. Navmesh: Build Paths

ADIM 3: Spawn Noktaları
────────────────────────
1. Player spawn: BP_PlayerStart (1 adet)
2. Bot spawn: BP_BotSpawnPoint (3-5 adet)
3. Her spawn point'e index ata (0, 1, 2...)

ADIM 4: Kameralar
────────────────────────
1. CameraActor ekle (5 adet)
2. Her birine tag ekle: "DemoCamera"
3. Pozisyonları ayarla (Bölüm 6.2'ye bak)

ADIM 5: Devriye Rotaları
────────────────────────
1. BP_PatrolRoute ekle (2 adet)
2. Patrol noktalarını spline ile çiz
3. Spawn point'lere ata

ADIM 6: Demo Controller
────────────────────────
1. BP_DemoController spawn
2. Auto Possess: Disabled
3. Referansları bağla

ADIM 7: UI Setup
────────────────────────
1. WBP_DemoHUD oluştur
2. Alt widget'ları ekle
3. Level Blueprint'te viewport'a ekle

ADIM 8: Lighting & Post Process
────────────────────────
1. Directional Light ekle
2. Sky Light ekle
3. Post Process Volume (debug görünürlük için)
   - Ambient Occlusion: Off
   - Bloom: Minimal

ADIM 9: Test
────────────────────────
1. PIE (Play In Editor) ile test
2. Tüm kısayolları dene
3. Her senaryoyu çalıştır
4. Performance kontrol (60 FPS hedef)
```

### 9.2 Checklist

```
PRE-DEMO CHECKLIST:
━━━━━━━━━━━━━━━━━━━

[ ] Level yüklendiğinde crash yok
[ ] NavMesh doğru build edilmiş
[ ] Tüm spawn noktaları çalışıyor
[ ] Kamera geçişleri smooth
[ ] Debug UI okunabilir
[ ] Zorluk slider çalışıyor
[ ] Pause/Resume çalışıyor
[ ] Zaman scale çalışıyor
[ ] RL server bağlantısı var (Senaryo 3 için)
[ ] Ekran çözünürlüğü uygun (1920x1080 önerilen)
[ ] Ses kapalı veya düşük (anlatım için)
[ ] Yedek kayıt hazır (video)
```

### 9.3 Olası Sorunlar ve Çözümler

| Sorun | Olası Neden | Çözüm |
|-------|-------------|-------|
| Bot hareket etmiyor | NavMesh yok | NavMesh Bounds Volume ekle, rebuild |
| Bot duvardan geçiyor | Collision yok | Collision channel kontrol et |
| UI görünmüyor | Widget eklenmemiş | Level BP'de AddToViewport kontrol |
| Kamera geçmiyor | Referans kayıp | Camera tag'lerini kontrol et |
| FPS düşük | Çok fazla debug draw | Debug detayını azalt |
| RL bot çalışmıyor | Server kapalı | Python server'ı başlat |

---

## Ekler

### Ek A: Renk Kodları

```
DEBUG RENK KODLARI:
━━━━━━━━━━━━━━━━━━━

Perception:
- Sight Range: Mavi (#4444FF)
- Hearing Range: Sarı (#FFFF44)
- Detected Enemy: Kırmızı (#FF4444)

Navigation:
- Path: Cyan (#44FFFF)
- Waypoint: Beyaz (#FFFFFF)

Bot Durumu:
- Alert: Kırmızı (#FF0000)
- Patrol: Yeşil (#00FF00)
- Taking Cover: Mavi (#0000FF)
- Fleeing: Sarı (#FFFF00)

Utility AI:
- Attack: #FF4444
- TakeCover: #4444FF
- Flee: #FFFF44
- Reload: #FF8844
- Patrol: #44FF44
- Flank: #FF44FF
```

### Ek B: Hakem Sunum Notları

```
SUNUM İPUÇLARI:
━━━━━━━━━━━━━━━

1. Demo öncesi:
   - Tüm sistemleri test et
   - Backup video hazırla
   - Notlarını hazırla

2. Sunum sırasında:
   - Yavaş ve net konuş
   - Her sistemi tek tek göster
   - Soruları not al

3. Teknik sorulara hazırlık:
   - "Bu algoritma neden seçildi?"
   - "Alternatifler neler?"
   - "Performance metrikleri nedir?"
   - "Gerçek oyunda nasıl ölçeklenecek?"

4. Vurgulanacak Ar-Ge noktaları:
   - Teknolojik belirsizlikler
   - Özgün çözümler
   - Literatür karşılaştırması
```

---

*Doküman Sonu*
