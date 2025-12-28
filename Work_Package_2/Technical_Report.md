# Yapay Zeka Destekli Bot Yazılımı MVP Teknik Raporu

**Proje:** Yapay Zeka Destekli Çok Oyunculu Coop TPS Oyunu  
**Doküman Versiyonu:** 1.0  
**Tarih:** 28.12.2025  
**Hazırlayan:** ttbilgin

---

## İçindekiler

1. [Giriş ve Kapsam](#1-giriş-ve-kapsam)
2. [Proje Kurulumu](#2-proje-kurulumu)
3. [Faz 1: AI Controller ve Temel Davranışlar](#3-faz-1-ai-controller-ve-temel-davranışlar)
4. [Faz 2: Behavior Tree Sistemi](#4-faz-2-behavior-tree-sistemi)
5. [Faz 3: Utility AI Entegrasyonu](#5-faz-3-utility-ai-entegrasyonu)
6. [Faz 4: Güçlendirmeli Öğrenme (RL) Prototipi](#6-faz-4-güçlendirmeli-öğrenme-rl-prototipi)
7. [Faz 5: Adaptif Zorluk Sistemi](#7-faz-5-adaptif-zorluk-sistemi)
8. [Faz 6: Inverse Kinematics (IK) Sistemi](#8-faz-6-inverse-kinematics-ik-sistemi)
9. [Faz 7: Test ve Debug Araçları](#9-faz-7-test-ve-debug-araçları)
10. [Demo Senaryoları](#10-demo-senaryoları)
11. [Proje Yapısı ve Dosya Organizasyonu](#11-proje-yapısı-ve-dosya-organizasyonu)

---

## 1. Giriş ve Kapsam

### 1.1 Amaç

Bu doküman, TÜBİTAK destekli "Yapay Zeka Destekli Çok Oyunculu Coop TPS Oyunu" projesinin İş Paketi 2 kapsamında geliştirilecek bot yazılımının Minimum Viable Product (MVP) versiyonunun teknik implementasyon rehberidir.

### 1.2 MVP Hedefleri

MVP aşağıdaki özellikleri içerecektir:

- Dürtü Tabanlı Yapay Zeka (Utility AI) ile dinamik karar verme
- Davranış Ağacı (Behavior Tree) ile hiyerarşik görev yönetimi
- Güçlendirmeli Öğrenme (Reinforcement Learning) prototipi
- İleri Seviye Inverse Kinematics (IK) ile gerçekçi animasyonlar
- Zorluk Seviyesine Göre Adaptif Düşman AI Modeli

### 1.3 Teknoloji Yığını

| Kategori | Teknoloji |
|----------|-----------|
| Oyun Motoru | Unreal Engine 5.3+ |
| Programlama | C++, Blueprints, Python |
| ML Framework | PyTorch, Stable-Baselines3 |
| Versiyon Kontrol | Git (Bitbucket) |
| Proje Yönetimi | Jira |

---

## 2. Proje Kurulumu

### 2.1 Unreal Engine Proje Yapılandırması

#### 2.1.1 Gerekli Pluginler

```
Plugins/
├── AIModule (Built-in)
├── NavigationSystem (Built-in)
├── GameplayTasks (Built-in)
├── GameplayAbilities (Optional)
└── LearningAgents (UE 5.3+, RL için)
```

#### 2.1.2 Build.cs Yapılandırması

```csharp
// BotAI.Build.cs
using UnrealBuildTool;

public class BotAI : ModuleRules
{
    public BotAI(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
        
        PublicDependencyModuleNames.AddRange(new string[] 
        { 
            "Core", 
            "CoreUObject", 
            "Engine", 
            "InputCore",
            "AIModule",
            "NavigationSystem",
            "GameplayTasks",
            "GameplayTags",
            "Sockets",
            "Networking"
        });
        
        PrivateDependencyModuleNames.AddRange(new string[] 
        {
            "AnimGraphRuntime"
        });
    }
}
```

#### 2.1.3 Config Dosyaları

```ini
; Config/DefaultGame.ini
[/Script/AIModule.AISystem]
bAllowControllersAsSubobjects=True
AcceptanceRadius=5.0
bFinishMoveOnGoalOverlap=True

[/Script/NavigationSystem.NavigationSystemV1]
bAutoCreateNavigationData=True
bAllowClientSideNavigation=False
```

### 2.2 Python Ortamı Kurulumu (RL için)

```bash
# Sanal ortam oluşturma
python -m venv bot_rl_env
source bot_rl_env/bin/activate  # Linux/Mac
# veya
bot_rl_env\Scripts\activate  # Windows

# Gerekli paketler
pip install torch>=2.0.0
pip install stable-baselines3>=2.0.0
pip install gymnasium>=0.29.0
pip install numpy>=1.24.0
pip install grpcio>=1.50.0
pip install tensorboard>=2.12.0
pip install protobuf>=4.21.0
```

#### 2.2.1 requirements.txt

```
torch>=2.0.0
stable-baselines3>=2.0.0
gymnasium>=0.29.0
numpy>=1.24.0
grpcio>=1.50.0
grpcio-tools>=1.50.0
tensorboard>=2.12.0
protobuf>=4.21.0
matplotlib>=3.7.0
pandas>=2.0.0
```

---

## 3. Faz 1: AI Controller ve Temel Davranışlar

### 3.1 Temel AI Controller Sınıfı

#### 3.1.1 Header Dosyası

```cpp
// Source/BotAI/Public/AI/BotAIController.h
#pragma once

#include "CoreMinimal.h"
#include "AIController.h"
#include "Perception/AIPerceptionTypes.h"
#include "BotAIController.generated.h"

class UBehaviorTreeComponent;
class UBlackboardComponent;
class UAIPerceptionComponent;
class UUtilityAIComponent;

UCLASS()
class BOTAI_API ABotAIController : public AAIController
{
    GENERATED_BODY()

public:
    ABotAIController();

    virtual void OnPossess(APawn* InPawn) override;
    virtual void OnUnPossess() override;
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // Perception
    UFUNCTION()
    void OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus);

    // Blackboard Helpers
    UFUNCTION(BlueprintCallable, Category = "AI")
    void SetTargetActor(AActor* NewTarget);

    UFUNCTION(BlueprintCallable, Category = "AI")
    AActor* GetTargetActor() const;

    UFUNCTION(BlueprintCallable, Category = "AI")
    void SetAlertState(bool bIsAlert);

protected:
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UBehaviorTreeComponent> BehaviorTreeComp;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UBlackboardComponent> BlackboardComp;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UAIPerceptionComponent> AIPerceptionComp;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
    TObjectPtr<UUtilityAIComponent> UtilityAIComp;

    // Configuration
    UPROPERTY(EditDefaultsOnly, Category = "AI")
    TObjectPtr<UBehaviorTree> BehaviorTree;

    UPROPERTY(EditDefaultsOnly, Category = "AI")
    TObjectPtr<UBlackboardData> BlackboardData;

    // Blackboard Keys
    UPROPERTY(EditDefaultsOnly, Category = "AI|Blackboard")
    FName TargetActorKey = "TargetActor";

    UPROPERTY(EditDefaultsOnly, Category = "AI|Blackboard")
    FName TargetLocationKey = "TargetLocation";

    UPROPERTY(EditDefaultsOnly, Category = "AI|Blackboard")
    FName IsAlertKey = "IsAlert";

    UPROPERTY(EditDefaultsOnly, Category = "AI|Blackboard")
    FName CurrentActionKey = "CurrentAction";

private:
    void SetupPerceptionSystem();
    void InitializeBlackboard();
};
```

#### 3.1.2 Implementasyon Dosyası

```cpp
// Source/BotAI/Private/AI/BotAIController.cpp
#include "AI/BotAIController.h"
#include "BehaviorTree/BehaviorTree.h"
#include "BehaviorTree/BehaviorTreeComponent.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "Perception/AIPerceptionComponent.h"
#include "Perception/AISenseConfig_Sight.h"
#include "Perception/AISenseConfig_Hearing.h"
#include "Perception/AISenseConfig_Damage.h"
#include "AI/UtilityAIComponent.h"

ABotAIController::ABotAIController()
{
    PrimaryActorTick.bCanEverTick = true;

    // Behavior Tree Component
    BehaviorTreeComp = CreateDefaultSubobject<UBehaviorTreeComponent>(
        TEXT("BehaviorTreeComponent"));
    
    // Blackboard Component
    BlackboardComp = CreateDefaultSubobject<UBlackboardComponent>(
        TEXT("BlackboardComponent"));
    
    // AI Perception Component
    AIPerceptionComp = CreateDefaultSubobject<UAIPerceptionComponent>(
        TEXT("AIPerceptionComponent"));
    
    // Utility AI Component
    UtilityAIComp = CreateDefaultSubobject<UUtilityAIComponent>(
        TEXT("UtilityAIComponent"));

    SetPerceptionComponent(*AIPerceptionComp);
}

void ABotAIController::BeginPlay()
{
    Super::BeginPlay();
    SetupPerceptionSystem();
}

void ABotAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    if (BlackboardData && BehaviorTree)
    {
        InitializeBlackboard();
        
        if (BehaviorTreeComp && BlackboardComp)
        {
            BehaviorTreeComp->StartTree(*BehaviorTree);
        }
    }
}

void ABotAIController::OnUnPossess()
{
    if (BehaviorTreeComp)
    {
        BehaviorTreeComp->StopTree();
    }
    
    Super::OnUnPossess();
}

void ABotAIController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // Utility AI değerlendirmesi (her frame yerine belirli aralıklarla)
    if (UtilityAIComp)
    {
        UtilityAIComp->EvaluateAndUpdateAction();
    }
}

void ABotAIController::SetupPerceptionSystem()
{
    if (!AIPerceptionComp) return;

    // Sight Configuration
    UAISenseConfig_Sight* SightConfig = NewObject<UAISenseConfig_Sight>(this);
    SightConfig->SightRadius = 2000.0f;
    SightConfig->LoseSightRadius = 2500.0f;
    SightConfig->PeripheralVisionAngleDegrees = 60.0f;
    SightConfig->SetMaxAge(5.0f);
    SightConfig->AutoSuccessRangeFromLastSeenLocation = 500.0f;
    SightConfig->DetectionByAffiliation.bDetectEnemies = true;
    SightConfig->DetectionByAffiliation.bDetectFriendlies = false;
    SightConfig->DetectionByAffiliation.bDetectNeutrals = false;
    AIPerceptionComp->ConfigureSense(*SightConfig);

    // Hearing Configuration
    UAISenseConfig_Hearing* HearingConfig = NewObject<UAISenseConfig_Hearing>(this);
    HearingConfig->HearingRange = 1500.0f;
    HearingConfig->SetMaxAge(3.0f);
    HearingConfig->DetectionByAffiliation.bDetectEnemies = true;
    HearingConfig->DetectionByAffiliation.bDetectFriendlies = true;
    HearingConfig->DetectionByAffiliation.bDetectNeutrals = true;
    AIPerceptionComp->ConfigureSense(*HearingConfig);

    // Damage Sense Configuration
    UAISenseConfig_Damage* DamageConfig = NewObject<UAISenseConfig_Damage>(this);
    DamageConfig->SetMaxAge(5.0f);
    AIPerceptionComp->ConfigureSense(*DamageConfig);

    // Dominant Sense
    AIPerceptionComp->SetDominantSense(SightConfig->GetSenseImplementation());

    // Bind perception update
    AIPerceptionComp->OnTargetPerceptionUpdated.AddDynamic(
        this, &ABotAIController::OnTargetPerceptionUpdated);
}

void ABotAIController::InitializeBlackboard()
{
    if (UseBlackboard(BlackboardData, BlackboardComp))
    {
        BlackboardComp->SetValueAsBool(IsAlertKey, false);
    }
}

void ABotAIController::OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus)
{
    if (!Actor || !BlackboardComp) return;

    // Düşman algılandığında
    if (Stimulus.WasSuccessfullySensed())
    {
        // Sadece oyuncu karakterlerini hedef al
        if (Actor->ActorHasTag(FName("Player")))
        {
            SetTargetActor(Actor);
            SetAlertState(true);
        }
    }
    else
    {
        // Hedef kaybedildi
        AActor* CurrentTarget = GetTargetActor();
        if (CurrentTarget == Actor)
        {
            // Son bilinen konumu kaydet
            BlackboardComp->SetValueAsVector(TargetLocationKey, 
                Stimulus.StimulusLocation);
        }
    }
}

void ABotAIController::SetTargetActor(AActor* NewTarget)
{
    if (BlackboardComp)
    {
        BlackboardComp->SetValueAsObject(TargetActorKey, NewTarget);
        
        if (NewTarget)
        {
            BlackboardComp->SetValueAsVector(TargetLocationKey, 
                NewTarget->GetActorLocation());
        }
    }
}

AActor* ABotAIController::GetTargetActor() const
{
    if (BlackboardComp)
    {
        return Cast<AActor>(
            BlackboardComp->GetValueAsObject(TargetActorKey));
    }
    return nullptr;
}

void ABotAIController::SetAlertState(bool bIsAlert)
{
    if (BlackboardComp)
    {
        BlackboardComp->SetValueAsBool(IsAlertKey, bIsAlert);
    }
}
```

### 3.2 Bot Character Sınıfı

#### 3.2.1 Header Dosyası

```cpp
// Source/BotAI/Public/Characters/BotCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "GenericTeamAgentInterface.h"
#include "BotCharacter.generated.h"

UENUM(BlueprintType)
enum class EBotDifficultyLevel : uint8
{
    Passenger    UMETA(DisplayName = "Yolcu"),
    Crew         UMETA(DisplayName = "Mürettebat"),
    ShipGuard    UMETA(DisplayName = "Gemi Muhafızı"),
    ShipPolice   UMETA(DisplayName = "Gemi Polisi"),
    Security     UMETA(DisplayName = "Koruma"),
    CoastGuard   UMETA(DisplayName = "Sahil Güvenlik"),
    SpecialForce UMETA(DisplayName = "Özel Kuvvetler")
};

USTRUCT(BlueprintType)
struct FBotStats
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float MaxHealth = 100.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CurrentHealth = 100.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float MovementSpeed = 400.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float AimAccuracy = 0.7f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ReactionTime = 0.3f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float AggressionLevel = 0.5f;
};

UCLASS()
class BOTAI_API ABotCharacter : public ACharacter, public IGenericTeamAgentInterface
{
    GENERATED_BODY()

public:
    ABotCharacter();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // IGenericTeamAgentInterface
    virtual FGenericTeamId GetGenericTeamId() const override;
    virtual void SetGenericTeamId(const FGenericTeamId& TeamID) override;

    // Combat
    UFUNCTION(BlueprintCallable, Category = "Combat")
    void FireWeapon();

    UFUNCTION(BlueprintCallable, Category = "Combat")
    void TakeDamage(float DamageAmount, AActor* DamageCauser);

    UFUNCTION(BlueprintCallable, Category = "Combat")
    bool IsAlive() const { return BotStats.CurrentHealth > 0.0f; }

    // Stats
    UFUNCTION(BlueprintCallable, Category = "Stats")
    FBotStats GetBotStats() const { return BotStats; }

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void SetDifficultyLevel(EBotDifficultyLevel NewLevel);

    UFUNCTION(BlueprintCallable, Category = "Stats")
    EBotDifficultyLevel GetDifficultyLevel() const { return DifficultyLevel; }

protected:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Bot")
    FBotStats BotStats;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Bot")
    EBotDifficultyLevel DifficultyLevel = EBotDifficultyLevel::Crew;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Team")
    FGenericTeamId TeamId;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
    TSubclassOf<AActor> ProjectileClass;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
    float FireRate = 0.5f;

private:
    float LastFireTime = 0.0f;
    
    void ApplyDifficultyModifiers();
};
```

#### 3.2.2 Implementasyon Dosyası

```cpp
// Source/BotAI/Private/Characters/BotCharacter.cpp
#include "Characters/BotCharacter.h"
#include "Components/CapsuleComponent.h"
#include "GameFramework/CharacterMovementComponent.h"

ABotCharacter::ABotCharacter()
{
    PrimaryActorTick.bCanEverTick = true;

    // Capsule setup
    GetCapsuleComponent()->InitCapsuleSize(42.0f, 96.0f);

    // Movement setup
    GetCharacterMovement()->bOrientRotationToMovement = true;
    GetCharacterMovement()->RotationRate = FRotator(0.0f, 540.0f, 0.0f);
    GetCharacterMovement()->MaxWalkSpeed = BotStats.MovementSpeed;

    // Default team (Enemy)
    TeamId = FGenericTeamId(1);

    // Tags
    Tags.Add(FName("Bot"));
}

void ABotCharacter::BeginPlay()
{
    Super::BeginPlay();
    ApplyDifficultyModifiers();
}

void ABotCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
}

FGenericTeamId ABotCharacter::GetGenericTeamId() const
{
    return TeamId;
}

void ABotCharacter::SetGenericTeamId(const FGenericTeamId& NewTeamID)
{
    TeamId = NewTeamID;
}

void ABotCharacter::SetDifficultyLevel(EBotDifficultyLevel NewLevel)
{
    DifficultyLevel = NewLevel;
    ApplyDifficultyModifiers();
}

void ABotCharacter::ApplyDifficultyModifiers()
{
    // Zorluk seviyesine göre bot istatistiklerini ayarla
    switch (DifficultyLevel)
    {
        case EBotDifficultyLevel::Passenger:
            BotStats.MaxHealth = 50.0f;
            BotStats.AimAccuracy = 0.2f;
            BotStats.ReactionTime = 1.0f;
            BotStats.AggressionLevel = 0.1f;
            BotStats.MovementSpeed = 200.0f;
            break;

        case EBotDifficultyLevel::Crew:
            BotStats.MaxHealth = 75.0f;
            BotStats.AimAccuracy = 0.4f;
            BotStats.ReactionTime = 0.7f;
            BotStats.AggressionLevel = 0.3f;
            BotStats.MovementSpeed = 300.0f;
            break;

        case EBotDifficultyLevel::ShipGuard:
            BotStats.MaxHealth = 100.0f;
            BotStats.AimAccuracy = 0.6f;
            BotStats.ReactionTime = 0.5f;
            BotStats.AggressionLevel = 0.5f;
            BotStats.MovementSpeed = 400.0f;
            break;

        case EBotDifficultyLevel::ShipPolice:
            BotStats.MaxHealth = 120.0f;
            BotStats.AimAccuracy = 0.7f;
            BotStats.ReactionTime = 0.4f;
            BotStats.AggressionLevel = 0.6f;
            BotStats.MovementSpeed = 450.0f;
            break;

        case EBotDifficultyLevel::Security:
            BotStats.MaxHealth = 150.0f;
            BotStats.AimAccuracy = 0.8f;
            BotStats.ReactionTime = 0.3f;
            BotStats.AggressionLevel = 0.7f;
            BotStats.MovementSpeed = 500.0f;
            break;

        case EBotDifficultyLevel::CoastGuard:
            BotStats.MaxHealth = 175.0f;
            BotStats.AimAccuracy = 0.85f;
            BotStats.ReactionTime = 0.25f;
            BotStats.AggressionLevel = 0.8f;
            BotStats.MovementSpeed = 525.0f;
            break;

        case EBotDifficultyLevel::SpecialForce:
            BotStats.MaxHealth = 200.0f;
            BotStats.AimAccuracy = 0.95f;
            BotStats.ReactionTime = 0.15f;
            BotStats.AggressionLevel = 0.9f;
            BotStats.MovementSpeed = 550.0f;
            break;
    }

    // Apply movement speed
    if (GetCharacterMovement())
    {
        GetCharacterMovement()->MaxWalkSpeed = BotStats.MovementSpeed;
    }

    // Reset current health
    BotStats.CurrentHealth = BotStats.MaxHealth;
}

void ABotCharacter::FireWeapon()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    
    if (CurrentTime - LastFireTime < FireRate)
        return;

    LastFireTime = CurrentTime;

    // Accuracy hesabı
    float AccuracyRoll = FMath::FRand();
    
    if (AccuracyRoll > BotStats.AimAccuracy)
    {
        // Iskalama - ateş etme ama isabetsiz
        return;
    }

    // Projectile spawn (basitleştirilmiş)
    if (ProjectileClass)
    {
        FVector MuzzleLocation = GetActorLocation() + GetActorForwardVector() * 100.0f;
        FRotator MuzzleRotation = GetActorRotation();
        
        GetWorld()->SpawnActor<AActor>(ProjectileClass, MuzzleLocation, MuzzleRotation);
    }
}

void ABotCharacter::TakeDamage(float DamageAmount, AActor* DamageCauser)
{
    BotStats.CurrentHealth = FMath::Max(0.0f, BotStats.CurrentHealth - DamageAmount);
    
    if (!IsAlive())
    {
        // Ölüm işlemi
        Destroy();
    }
}
```

---

## 4. Faz 2: Behavior Tree Sistemi

### 4.1 Blackboard Data Asset

Unreal Editor'da oluşturulacak Blackboard asset için key tanımları:

| Key Name | Type | Description |
|----------|------|-------------|
| TargetActor | Object (AActor) | Mevcut hedef |
| TargetLocation | Vector | Hedefin son bilinen konumu |
| IsAlert | Bool | Alarm durumu |
| CurrentAction | Enum | Aktif aksiyon (Utility AI'dan) |
| PatrolIndex | Int | Devriye noktası indeksi |
| CoverLocation | Vector | Siper pozisyonu |
| HealthPercentage | Float | Sağlık yüzdesi |
| AmmoCount | Int | Mermi sayısı |
| ThreatLevel | Float | Tehdit seviyesi (0-1) |

### 4.2 Custom Behavior Tree Tasks

#### 4.2.1 BTTask_FindCover - Siper Bulma

```cpp
// Source/BotAI/Public/AI/Tasks/BTTask_FindCover.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTTaskNode.h"
#include "BTTask_FindCover.generated.h"

UCLASS()
class BOTAI_API UBTTask_FindCover : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_FindCover();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
        uint8* NodeMemory) override;

protected:
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector CoverLocationKey;

    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector TargetActorKey;

    UPROPERTY(EditAnywhere, Category = "Cover")
    float SearchRadius = 1000.0f;

    UPROPERTY(EditAnywhere, Category = "Cover")
    float MinDistanceFromThreat = 300.0f;

private:
    bool FindBestCoverLocation(AActor* Bot, AActor* Threat, FVector& OutLocation);
};
```

```cpp
// Source/BotAI/Private/AI/Tasks/BTTask_FindCover.cpp
#include "AI/Tasks/BTTask_FindCover.h"
#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "NavigationSystem.h"
#include "EnvironmentQuery/EnvQueryManager.h"

UBTTask_FindCover::UBTTask_FindCover()
{
    NodeName = "Find Cover Position";
    
    CoverLocationKey.AddVectorFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTTask_FindCover, CoverLocationKey));
    TargetActorKey.AddObjectFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTTask_FindCover, TargetActorKey), AActor::StaticClass());
}

EBTNodeResult::Type UBTTask_FindCover::ExecuteTask(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    APawn* Bot = AIController->GetPawn();
    if (!Bot) return EBTNodeResult::Failed;

    UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent();
    if (!BlackboardComp) return EBTNodeResult::Failed;

    AActor* Threat = Cast<AActor>(
        BlackboardComp->GetValueAsObject(TargetActorKey.SelectedKeyName));

    FVector CoverLocation;
    if (FindBestCoverLocation(Bot, Threat, CoverLocation))
    {
        BlackboardComp->SetValueAsVector(CoverLocationKey.SelectedKeyName, 
            CoverLocation);
        return EBTNodeResult::Succeeded;
    }

    return EBTNodeResult::Failed;
}

bool UBTTask_FindCover::FindBestCoverLocation(AActor* Bot, AActor* Threat, 
    FVector& OutLocation)
{
    if (!Bot) return false;

    UNavigationSystemV1* NavSys = UNavigationSystemV1::GetCurrent(Bot->GetWorld());
    if (!NavSys) return false;

    FVector BotLocation = Bot->GetActorLocation();
    FVector ThreatLocation = Threat ? Threat->GetActorLocation() : BotLocation;
    FVector DirectionFromThreat = (BotLocation - ThreatLocation).GetSafeNormal();

    // Tehditin arkasına doğru siper ara
    TArray<FVector> CandidateLocations;
    const int32 NumSamples = 16;
    
    for (int32 i = 0; i < NumSamples; ++i)
    {
        float Angle = (2.0f * PI * i) / NumSamples;
        FVector Offset = FVector(
            FMath::Cos(Angle) * SearchRadius,
            FMath::Sin(Angle) * SearchRadius,
            0.0f
        );

        FVector TestLocation = BotLocation + Offset;
        FNavLocation NavLocation;

        if (NavSys->ProjectPointToNavigation(TestLocation, NavLocation))
        {
            // Tehdide çok yakın mı kontrol et
            float DistanceToThreat = FVector::Dist(NavLocation.Location, ThreatLocation);
            
            if (DistanceToThreat >= MinDistanceFromThreat)
            {
                // Line of sight kontrolü (basitleştirilmiş)
                FHitResult HitResult;
                FCollisionQueryParams QueryParams;
                QueryParams.AddIgnoredActor(Bot);

                bool bHasLineOfSight = Bot->GetWorld()->LineTraceSingleByChannel(
                    HitResult,
                    NavLocation.Location + FVector(0, 0, 50),
                    ThreatLocation + FVector(0, 0, 50),
                    ECC_Visibility,
                    QueryParams
                );

                // Tehditten görünmüyorsa iyi bir siper
                if (bHasLineOfSight && HitResult.bBlockingHit)
                {
                    CandidateLocations.Add(NavLocation.Location);
                }
            }
        }
    }

    if (CandidateLocations.Num() > 0)
    {
        // En yakın siperi seç
        float BestDistance = MAX_FLT;
        for (const FVector& Candidate : CandidateLocations)
        {
            float Distance = FVector::Dist(BotLocation, Candidate);
            if (Distance < BestDistance)
            {
                BestDistance = Distance;
                OutLocation = Candidate;
            }
        }
        return true;
    }

    return false;
}
```

#### 4.2.2 BTTask_Attack - Saldırı

```cpp
// Source/BotAI/Public/AI/Tasks/BTTask_Attack.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTTaskNode.h"
#include "BTTask_Attack.generated.h"

UCLASS()
class BOTAI_API UBTTask_Attack : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_Attack();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
        uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, 
        float DeltaSeconds) override;

protected:
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector TargetActorKey;

    UPROPERTY(EditAnywhere, Category = "Attack")
    float AttackDuration = 2.0f;

    UPROPERTY(EditAnywhere, Category = "Attack")
    float AttackRange = 1500.0f;

private:
    float ElapsedTime = 0.0f;
};
```

```cpp
// Source/BotAI/Private/AI/Tasks/BTTask_Attack.cpp
#include "AI/Tasks/BTTask_Attack.h"
#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "Characters/BotCharacter.h"

UBTTask_Attack::UBTTask_Attack()
{
    NodeName = "Attack Target";
    bNotifyTick = true;

    TargetActorKey.AddObjectFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTTask_Attack, TargetActorKey), AActor::StaticClass());
}

EBTNodeResult::Type UBTTask_Attack::ExecuteTask(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return EBTNodeResult::Failed;

    UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent();
    if (!BlackboardComp) return EBTNodeResult::Failed;

    AActor* Target = Cast<AActor>(
        BlackboardComp->GetValueAsObject(TargetActorKey.SelectedKeyName));
    
    if (!Target) return EBTNodeResult::Failed;

    APawn* BotPawn = AIController->GetPawn();
    if (!BotPawn) return EBTNodeResult::Failed;

    // Menzil kontrolü
    float Distance = FVector::Dist(BotPawn->GetActorLocation(), 
        Target->GetActorLocation());
    
    if (Distance > AttackRange)
    {
        return EBTNodeResult::Failed;
    }

    // Hedefe dön
    FVector Direction = (Target->GetActorLocation() - 
        BotPawn->GetActorLocation()).GetSafeNormal();
    FRotator TargetRotation = Direction.Rotation();
    BotPawn->SetActorRotation(TargetRotation);

    ElapsedTime = 0.0f;
    return EBTNodeResult::InProgress;
}

void UBTTask_Attack::TickTask(UBehaviorTreeComponent& OwnerComp, 
    uint8* NodeMemory, float DeltaSeconds)
{
    ElapsedTime += DeltaSeconds;

    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Failed);
        return;
    }

    ABotCharacter* BotCharacter = Cast<ABotCharacter>(AIController->GetPawn());
    if (BotCharacter)
    {
        // Ateş et
        BotCharacter->FireWeapon();
    }

    // Saldırı süresi doldu mu?
    if (ElapsedTime >= AttackDuration)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}
```

#### 4.2.3 BTTask_Patrol - Devriye

```cpp
// Source/BotAI/Public/AI/Tasks/BTTask_Patrol.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTTaskNode.h"
#include "BTTask_Patrol.generated.h"

UCLASS()
class BOTAI_API UBTTask_Patrol : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UBTTask_Patrol();

    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, 
        uint8* NodeMemory) override;

protected:
    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector PatrolLocationKey;

    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector PatrolIndexKey;

    UPROPERTY(EditAnywhere, Category = "Patrol")
    TArray<FVector> PatrolPoints;

    UPROPERTY(EditAnywhere, Category = "Patrol")
    bool bLoopPatrol = true;

    UPROPERTY(EditAnywhere, Category = "Patrol")
    float PatrolPointRadius = 100.0f;
};
```

```cpp
// Source/BotAI/Private/AI/Tasks/BTTask_Patrol.cpp
#include "AI/Tasks/BTTask_Patrol.h"
#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"

UBTTask_Patrol::UBTTask_Patrol()
{
    NodeName = "Get Next Patrol Point";
    
    PatrolLocationKey.AddVectorFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTTask_Patrol, PatrolLocationKey));
    PatrolIndexKey.AddIntFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTTask_Patrol, PatrolIndexKey));
}

EBTNodeResult::Type UBTTask_Patrol::ExecuteTask(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    if (PatrolPoints.Num() == 0)
    {
        return EBTNodeResult::Failed;
    }

    UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent();
    if (!BlackboardComp) return EBTNodeResult::Failed;

    int32 CurrentIndex = BlackboardComp->GetValueAsInt(PatrolIndexKey.SelectedKeyName);
    
    // Geçerli indeks kontrolü
    if (CurrentIndex >= PatrolPoints.Num())
    {
        if (bLoopPatrol)
        {
            CurrentIndex = 0;
        }
        else
        {
            return EBTNodeResult::Failed;
        }
    }

    // Patrol noktasını ayarla
    FVector PatrolLocation = PatrolPoints[CurrentIndex];
    BlackboardComp->SetValueAsVector(PatrolLocationKey.SelectedKeyName, PatrolLocation);
    
    // Sonraki indeksi ayarla
    int32 NextIndex = CurrentIndex + 1;
    if (bLoopPatrol && NextIndex >= PatrolPoints.Num())
    {
        NextIndex = 0;
    }
    BlackboardComp->SetValueAsInt(PatrolIndexKey.SelectedKeyName, NextIndex);

    return EBTNodeResult::Succeeded;
}
```

### 4.3 Custom Decorators

#### 4.3.1 BTDecorator_HasLineOfSight - Görüş Hattı Kontrolü

```cpp
// Source/BotAI/Public/AI/Decorators/BTDecorator_HasLineOfSight.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTDecorator.h"
#include "BTDecorator_HasLineOfSight.generated.h"

UCLASS()
class BOTAI_API UBTDecorator_HasLineOfSight : public UBTDecorator
{
    GENERATED_BODY()

public:
    UBTDecorator_HasLineOfSight();

protected:
    virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, 
        uint8* NodeMemory) const override;

    UPROPERTY(EditAnywhere, Category = "Blackboard")
    FBlackboardKeySelector TargetActorKey;

    UPROPERTY(EditAnywhere, Category = "LineOfSight")
    float MaxDistance = 2000.0f;
};
```

```cpp
// Source/BotAI/Private/AI/Decorators/BTDecorator_HasLineOfSight.cpp
#include "AI/Decorators/BTDecorator_HasLineOfSight.h"
#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"

UBTDecorator_HasLineOfSight::UBTDecorator_HasLineOfSight()
{
    NodeName = "Has Line of Sight";

    TargetActorKey.AddObjectFilter(this, GET_MEMBER_NAME_CHECKED(
        UBTDecorator_HasLineOfSight, TargetActorKey), AActor::StaticClass());
}

bool UBTDecorator_HasLineOfSight::CalculateRawConditionValue(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return false;

    APawn* BotPawn = AIController->GetPawn();
    if (!BotPawn) return false;

    UBlackboardComponent* BlackboardComp = OwnerComp.GetBlackboardComponent();
    if (!BlackboardComp) return false;

    AActor* Target = Cast<AActor>(
        BlackboardComp->GetValueAsObject(TargetActorKey.SelectedKeyName));
    
    if (!Target) return false;

    // Mesafe kontrolü
    float Distance = FVector::Dist(BotPawn->GetActorLocation(), 
        Target->GetActorLocation());
    
    if (Distance > MaxDistance) return false;

    // Line trace
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(BotPawn);

    FVector StartLocation = BotPawn->GetActorLocation() + FVector(0, 0, 50);
    FVector EndLocation = Target->GetActorLocation() + FVector(0, 0, 50);

    bool bHit = BotPawn->GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        ECC_Visibility,
        QueryParams
    );

    // Hedefe isabet ettiyse görüş hattı var
    if (bHit && HitResult.GetActor() == Target)
    {
        return true;
    }

    return !bHit; // Hiçbir şeye çarpmadıysa da görüş hattı var
}
```

#### 4.3.2 BTDecorator_CheckHealth - Sağlık Kontrolü

```cpp
// Source/BotAI/Public/AI/Decorators/BTDecorator_CheckHealth.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTDecorator.h"
#include "BTDecorator_CheckHealth.generated.h"

UENUM(BlueprintType)
enum class EHealthCheckType : uint8
{
    LessThan,
    GreaterThan,
    Equals
};

UCLASS()
class BOTAI_API UBTDecorator_CheckHealth : public UBTDecorator
{
    GENERATED_BODY()

public:
    UBTDecorator_CheckHealth();

protected:
    virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, 
        uint8* NodeMemory) const override;

    UPROPERTY(EditAnywhere, Category = "Health")
    EHealthCheckType CheckType = EHealthCheckType::LessThan;

    UPROPERTY(EditAnywhere, Category = "Health", meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float HealthThreshold = 0.3f;
};
```

```cpp
// Source/BotAI/Private/AI/Decorators/BTDecorator_CheckHealth.cpp
#include "AI/Decorators/BTDecorator_CheckHealth.h"
#include "AIController.h"
#include "Characters/BotCharacter.h"

UBTDecorator_CheckHealth::UBTDecorator_CheckHealth()
{
    NodeName = "Check Health";
}

bool UBTDecorator_CheckHealth::CalculateRawConditionValue(
    UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (!AIController) return false;

    ABotCharacter* BotCharacter = Cast<ABotCharacter>(AIController->GetPawn());
    if (!BotCharacter) return false;

    FBotStats Stats = BotCharacter->GetBotStats();
    float HealthPercentage = Stats.CurrentHealth / Stats.MaxHealth;

    switch (CheckType)
    {
        case EHealthCheckType::LessThan:
            return HealthPercentage < HealthThreshold;
        
        case EHealthCheckType::GreaterThan:
            return HealthPercentage > HealthThreshold;
        
        case EHealthCheckType::Equals:
            return FMath::IsNearlyEqual(HealthPercentage, HealthThreshold, 0.05f);
    }

    return false;
}
```

---

## 5. Faz 3: Utility AI Entegrasyonu

### 5.1 Utility AI Component

#### 5.1.1 Temel Yapılar

```cpp
// Source/BotAI/Public/AI/UtilityAI/UtilityAITypes.h
#pragma once

#include "CoreMinimal.h"
#include "UtilityAITypes.generated.h"

UENUM(BlueprintType)
enum class EBotAction : uint8
{
    None,
    Attack,
    TakeCover,
    Flee,
    Reload,
    Patrol,
    Investigate,
    SupportTeammate,
    Flank
};

UENUM(BlueprintType)
enum class EConsiderationType : uint8
{
    Health,
    Ammo,
    DistanceToTarget,
    ThreatLevel,
    TeammateProximity,
    CoverAvailability,
    TargetHealth,
    LineOfSight
};

USTRUCT(BlueprintType)
struct FUtilityAction
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EBotAction ActionType = EBotAction::None;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float BaseScore = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<EConsiderationType> Considerations;

    // Hesaplanmış final skor
    float FinalScore = 0.0f;
};

USTRUCT(BlueprintType)
struct FConsiderationData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EConsiderationType Type;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float InputMin = 0.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float InputMax = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    UCurveFloat* ResponseCurve = nullptr;

    // Manuel curve parametreleri (ResponseCurve yoksa kullanılır)
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CurveSlope = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CurveExponent = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CurveXShift = 0.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CurveYShift = 0.0f;
};
```

#### 5.1.2 Utility AI Component

```cpp
// Source/BotAI/Public/AI/UtilityAI/UtilityAIComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "UtilityAITypes.h"
#include "UtilityAIComponent.generated.h"

class ABotCharacter;
class ABotAIController;

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnActionChanged, EBotAction, NewAction);

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class BOTAI_API UUtilityAIComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UUtilityAIComponent();

    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
        FActorComponentTickFunction* ThisTickFunction) override;

    // Ana değerlendirme fonksiyonu
    UFUNCTION(BlueprintCallable, Category = "Utility AI")
    void EvaluateAndUpdateAction();

    // Mevcut aksiyonu al
    UFUNCTION(BlueprintCallable, Category = "Utility AI")
    EBotAction GetCurrentAction() const { return CurrentAction; }

    // Aksiyon skorlarını al (debug için)
    UFUNCTION(BlueprintCallable, Category = "Utility AI")
    TMap<EBotAction, float> GetActionScores() const { return ActionScores; }

    // Delegate
    UPROPERTY(BlueprintAssignable, Category = "Utility AI")
    FOnActionChanged OnActionChanged;

protected:
    // Aksiyonlar ve ayarları
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Utility AI")
    TArray<FUtilityAction> AvailableActions;

    // Consideration ayarları
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Utility AI")
    TMap<EConsiderationType, FConsiderationData> ConsiderationConfigs;

    // Değerlendirme sıklığı
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Utility AI")
    float EvaluationInterval = 0.2f;

    // Debug
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Utility AI|Debug")
    bool bShowDebug = false;

private:
    // Mevcut aksiyon
    EBotAction CurrentAction = EBotAction::None;

    // Aksiyon skorları
    TMap<EBotAction, float> ActionScores;

    // Zamanlama
    float TimeSinceLastEvaluation = 0.0f;

    // Referanslar
    UPROPERTY()
    ABotCharacter* OwnerCharacter;

    UPROPERTY()
    ABotAIController* OwnerController;

    // Yardımcı fonksiyonlar
    float EvaluateConsideration(EConsiderationType Type);
    float GetRawInputValue(EConsiderationType Type);
    float ApplyResponseCurve(float Input, const FConsiderationData& Config);
    float CalculateActionScore(const FUtilityAction& Action);
    EBotAction SelectBestAction();
    void SetupDefaultActions();
    void SetupDefaultConsiderations();
    void DrawDebugInfo();
};
```

#### 5.1.3 Utility AI Component Implementation

```cpp
// Source/BotAI/Private/AI/UtilityAI/UtilityAIComponent.cpp
#include "AI/UtilityAI/UtilityAIComponent.h"
#include "Characters/BotCharacter.h"
#include "AI/BotAIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "DrawDebugHelpers.h"

UUtilityAIComponent::UUtilityAIComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UUtilityAIComponent::BeginPlay()
{
    Super::BeginPlay();

    OwnerCharacter = Cast<ABotCharacter>(GetOwner());
    if (OwnerCharacter)
    {
        OwnerController = Cast<ABotAIController>(OwnerCharacter->GetController());
    }

    // Default ayarları yükle
    if (AvailableActions.Num() == 0)
    {
        SetupDefaultActions();
    }

    if (ConsiderationConfigs.Num() == 0)
    {
        SetupDefaultConsiderations();
    }
}

void UUtilityAIComponent::TickComponent(float DeltaTime, ELevelTick TickType, 
    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    TimeSinceLastEvaluation += DeltaTime;

    if (TimeSinceLastEvaluation >= EvaluationInterval)
    {
        EvaluateAndUpdateAction();
        TimeSinceLastEvaluation = 0.0f;
    }

    if (bShowDebug)
    {
        DrawDebugInfo();
    }
}

void UUtilityAIComponent::SetupDefaultActions()
{
    // Attack Action
    FUtilityAction AttackAction;
    AttackAction.ActionType = EBotAction::Attack;
    AttackAction.BaseScore = 1.0f;
    AttackAction.Considerations.Add(EConsiderationType::LineOfSight);
    AttackAction.Considerations.Add(EConsiderationType::Ammo);
    AttackAction.Considerations.Add(EConsiderationType::DistanceToTarget);
    AvailableActions.Add(AttackAction);

    // Take Cover Action
    FUtilityAction CoverAction;
    CoverAction.ActionType = EBotAction::TakeCover;
    CoverAction.BaseScore = 1.0f;
    CoverAction.Considerations.Add(EConsiderationType::Health);
    CoverAction.Considerations.Add(EConsiderationType::ThreatLevel);
    CoverAction.Considerations.Add(EConsiderationType::CoverAvailability);
    AvailableActions.Add(CoverAction);

    // Flee Action
    FUtilityAction FleeAction;
    FleeAction.ActionType = EBotAction::Flee;
    FleeAction.BaseScore = 0.8f;
    FleeAction.Considerations.Add(EConsiderationType::Health);
    FleeAction.Considerations.Add(EConsiderationType::ThreatLevel);
    FleeAction.Considerations.Add(EConsiderationType::Ammo);
    AvailableActions.Add(FleeAction);

    // Reload Action
    FUtilityAction ReloadAction;
    ReloadAction.ActionType = EBotAction::Reload;
    ReloadAction.BaseScore = 0.9f;
    ReloadAction.Considerations.Add(EConsiderationType::Ammo);
    ReloadAction.Considerations.Add(EConsiderationType::CoverAvailability);
    AvailableActions.Add(ReloadAction);

    // Patrol Action
    FUtilityAction PatrolAction;
    PatrolAction.ActionType = EBotAction::Patrol;
    PatrolAction.BaseScore = 0.5f;
    PatrolAction.Considerations.Add(EConsiderationType::ThreatLevel);
    AvailableActions.Add(PatrolAction);

    // Flank Action
    FUtilityAction FlankAction;
    FlankAction.ActionType = EBotAction::Flank;
    FlankAction.BaseScore = 0.7f;
    FlankAction.Considerations.Add(EConsiderationType::TeammateProximity);
    FlankAction.Considerations.Add(EConsiderationType::Health);
    FlankAction.Considerations.Add(EConsiderationType::DistanceToTarget);
    AvailableActions.Add(FlankAction);
}

void UUtilityAIComponent::SetupDefaultConsiderations()
{
    // Health Consideration (düşük sağlık = yüksek değer)
    FConsiderationData HealthConfig;
    HealthConfig.Type = EConsiderationType::Health;
    HealthConfig.InputMin = 0.0f;
    HealthConfig.InputMax = 1.0f;
    HealthConfig.CurveSlope = -1.0f; // Ters orantılı
    HealthConfig.CurveExponent = 2.0f;
    HealthConfig.CurveYShift = 1.0f;
    ConsiderationConfigs.Add(EConsiderationType::Health, HealthConfig);

    // Ammo Consideration
    FConsiderationData AmmoConfig;
    AmmoConfig.Type = EConsiderationType::Ammo;
    AmmoConfig.InputMin = 0.0f;
    AmmoConfig.InputMax = 1.0f;
    AmmoConfig.CurveSlope = 1.0f;
    AmmoConfig.CurveExponent = 1.0f;
    ConsiderationConfigs.Add(EConsiderationType::Ammo, AmmoConfig);

    // Distance to Target (yakın = yüksek değer saldırı için)
    FConsiderationData DistanceConfig;
    DistanceConfig.Type = EConsiderationType::DistanceToTarget;
    DistanceConfig.InputMin = 0.0f;
    DistanceConfig.InputMax = 2000.0f;
    DistanceConfig.CurveSlope = -1.0f;
    DistanceConfig.CurveExponent = 0.5f;
    DistanceConfig.CurveYShift = 1.0f;
    ConsiderationConfigs.Add(EConsiderationType::DistanceToTarget, DistanceConfig);

    // Threat Level
    FConsiderationData ThreatConfig;
    ThreatConfig.Type = EConsiderationType::ThreatLevel;
    ThreatConfig.InputMin = 0.0f;
    ThreatConfig.InputMax = 1.0f;
    ThreatConfig.CurveSlope = 1.0f;
    ThreatConfig.CurveExponent = 1.5f;
    ConsiderationConfigs.Add(EConsiderationType::ThreatLevel, ThreatConfig);

    // Line of Sight (binary)
    FConsiderationData LOSConfig;
    LOSConfig.Type = EConsiderationType::LineOfSight;
    LOSConfig.InputMin = 0.0f;
    LOSConfig.InputMax = 1.0f;
    LOSConfig.CurveSlope = 1.0f;
    LOSConfig.CurveExponent = 1.0f;
    ConsiderationConfigs.Add(EConsiderationType::LineOfSight, LOSConfig);

    // Cover Availability
    FConsiderationData CoverConfig;
    CoverConfig.Type = EConsiderationType::CoverAvailability;
    CoverConfig.InputMin = 0.0f;
    CoverConfig.InputMax = 1.0f;
    CoverConfig.CurveSlope = 1.0f;
    CoverConfig.CurveExponent = 1.0f;
    ConsiderationConfigs.Add(EConsiderationType::CoverAvailability, CoverConfig);

    // Teammate Proximity
    FConsiderationData TeammateConfig;
    TeammateConfig.Type = EConsiderationType::TeammateProximity;
    TeammateConfig.InputMin = 0.0f;
    TeammateConfig.InputMax = 1000.0f;
    TeammateConfig.CurveSlope = 1.0f;
    TeammateConfig.CurveExponent = 0.5f;
    ConsiderationConfigs.Add(EConsiderationType::TeammateProximity, TeammateConfig);
}

float UUtilityAIComponent::GetRawInputValue(EConsiderationType Type)
{
    if (!OwnerCharacter) return 0.0f;

    FBotStats Stats = OwnerCharacter->GetBotStats();
    AActor* Target = OwnerController ? OwnerController->GetTargetActor() : nullptr;

    switch (Type)
    {
        case EConsiderationType::Health:
            return Stats.CurrentHealth / Stats.MaxHealth;

        case EConsiderationType::Ammo:
            // Basit bir ammo sistemi varsayalım (0-1 arası)
            return 0.7f; // Placeholder

        case EConsiderationType::DistanceToTarget:
            if (Target)
            {
                return FVector::Dist(OwnerCharacter->GetActorLocation(), 
                    Target->GetActorLocation());
            }
            return 9999.0f;

        case EConsiderationType::ThreatLevel:
            // Blackboard'dan tehdit seviyesi
            if (OwnerController)
            {
                UBlackboardComponent* BB = OwnerController->GetBlackboardComponent();
                if (BB)
                {
                    return BB->GetValueAsFloat(FName("ThreatLevel"));
                }
            }
            return 0.0f;

        case EConsiderationType::LineOfSight:
            if (Target && OwnerCharacter)
            {
                FHitResult Hit;
                FCollisionQueryParams Params;
                Params.AddIgnoredActor(OwnerCharacter);
                
                bool bHit = OwnerCharacter->GetWorld()->LineTraceSingleByChannel(
                    Hit,
                    OwnerCharacter->GetActorLocation() + FVector(0, 0, 50),
                    Target->GetActorLocation() + FVector(0, 0, 50),
                    ECC_Visibility,
                    Params
                );
                
                return (!bHit || Hit.GetActor() == Target) ? 1.0f : 0.0f;
            }
            return 0.0f;

        case EConsiderationType::CoverAvailability:
            // Basitleştirilmiş - her zaman siper var varsay
            return 0.8f;

        case EConsiderationType::TeammateProximity:
            // Yakın takım arkadaşı sayısına göre
            return 0.5f; // Placeholder
    }

    return 0.0f;
}

float UUtilityAIComponent::ApplyResponseCurve(float Input, const FConsiderationData& Config)
{
    // Normalize input
    float NormalizedInput = FMath::Clamp(
        (Input - Config.InputMin) / (Config.InputMax - Config.InputMin),
        0.0f, 1.0f
    );

    // Custom response curve kullanılıyorsa
    if (Config.ResponseCurve)
    {
        return Config.ResponseCurve->GetFloatValue(NormalizedInput);
    }

    // Manuel curve: y = m * (x - x_shift)^k + y_shift
    float ShiftedX = NormalizedInput - Config.CurveXShift;
    float Result = Config.CurveSlope * FMath::Pow(FMath::Abs(ShiftedX), 
        Config.CurveExponent) + Config.CurveYShift;
    
    if (ShiftedX < 0 && FMath::Fmod(Config.CurveExponent, 2.0f) != 0)
    {
        Result = Config.CurveSlope * -FMath::Pow(FMath::Abs(ShiftedX), 
            Config.CurveExponent) + Config.CurveYShift;
    }

    return FMath::Clamp(Result, 0.0f, 1.0f);
}

float UUtilityAIComponent::EvaluateConsideration(EConsiderationType Type)
{
    if (!ConsiderationConfigs.Contains(Type))
    {
        return 0.5f; // Default değer
    }

    const FConsiderationData& Config = ConsiderationConfigs[Type];
    float RawValue = GetRawInputValue(Type);
    return ApplyResponseCurve(RawValue, Config);
}

float UUtilityAIComponent::CalculateActionScore(const FUtilityAction& Action)
{
    if (Action.Considerations.Num() == 0)
    {
        return Action.BaseScore;
    }

    // Geometric mean - tüm consideration skorlarının çarpımı
    float Score = Action.BaseScore;
    
    for (EConsiderationType ConsType : Action.Considerations)
    {
        float ConsScore = EvaluateConsideration(ConsType);
        Score *= ConsScore;
    }

    // Compensation factor (Dave Mark'ın önerisi)
    // Daha fazla consideration'a sahip aksiyonlar dezavantajlı olmasın
    float ModificationFactor = 1.0f - (1.0f / Action.Considerations.Num());
    float MakeUpValue = (1.0f - Score) * ModificationFactor;
    Score += MakeUpValue * Score;

    return Score;
}

EBotAction UUtilityAIComponent::SelectBestAction()
{
    ActionScores.Empty();
    
    EBotAction BestAction = EBotAction::None;
    float BestScore = -1.0f;

    for (FUtilityAction& Action : AvailableActions)
    {
        Action.FinalScore = CalculateActionScore(Action);
        ActionScores.Add(Action.ActionType, Action.FinalScore);

        if (Action.FinalScore > BestScore)
        {
            BestScore = Action.FinalScore;
            BestAction = Action.ActionType;
        }
    }

    return BestAction;
}

void UUtilityAIComponent::EvaluateAndUpdateAction()
{
    EBotAction NewAction = SelectBestAction();

    if (NewAction != CurrentAction)
    {
        EBotAction OldAction = CurrentAction;
        CurrentAction = NewAction;
        OnActionChanged.Broadcast(NewAction);

        // Blackboard'u güncelle
        if (OwnerController)
        {
            UBlackboardComponent* BB = OwnerController->GetBlackboardComponent();
            if (BB)
            {
                BB->SetValueAsEnum(FName("CurrentAction"), 
                    static_cast<uint8>(NewAction));
            }
        }
    }
}

void UUtilityAIComponent::DrawDebugInfo()
{
    if (!OwnerCharacter) return;

    FVector Location = OwnerCharacter->GetActorLocation() + FVector(0, 0, 150);

    // Mevcut aksiyonu göster
    FString ActionName = UEnum::GetValueAsString(CurrentAction);
    DrawDebugString(GetWorld(), Location, ActionName, nullptr, 
        FColor::Yellow, 0.0f, true, 1.2f);

    // Aksiyon skorlarını göster
    float YOffset = 20.0f;
    for (const auto& Pair : ActionScores)
    {
        FString ScoreText = FString::Printf(TEXT("%s: %.2f"), 
            *UEnum::GetValueAsString(Pair.Key), Pair.Value);
        
        FColor ScoreColor = (Pair.Key == CurrentAction) ? FColor::Green : FColor::White;
        
        DrawDebugString(GetWorld(), Location + FVector(0, 0, -YOffset), 
            ScoreText, nullptr, ScoreColor, 0.0f, true, 0.8f);
        
        YOffset += 15.0f;
    }
}
```

---

## 6. Faz 4: Güçlendirmeli Öğrenme (RL) Prototipi

### 6.1 Unreal Engine - Python Bridge

#### 6.1.1 gRPC Proto Tanımı

```protobuf
// Protos/bot_ai.proto
syntax = "proto3";

package botai;

service BotAIService {
    // Oyun durumunu al, aksiyon döndür
    rpc GetAction(GameState) returns (BotAction);
    
    // Eğitim için deneyim gönder
    rpc SendExperience(Experience) returns (Acknowledgment);
    
    // Eğitimi başlat/durdur
    rpc ControlTraining(TrainingCommand) returns (TrainingStatus);
}

message GameState {
    // Bot durumu
    float bot_health = 1;
    float bot_ammo = 2;
    repeated float bot_position = 3;  // x, y, z
    repeated float bot_rotation = 4;  // pitch, yaw, roll
    
    // Hedef durumu
    bool has_target = 5;
    repeated float target_position = 6;
    float target_health = 7;
    float distance_to_target = 8;
    bool has_line_of_sight = 9;
    
    // Çevre durumu
    repeated float nearby_cover_positions = 10;  // Flattened array
    int32 num_nearby_enemies = 11;
    int32 num_nearby_allies = 12;
    
    // Oyun durumu
    float game_time = 13;
    int32 current_phase = 14;  // 0: Boarding, 1: Searching, 2: Extraction
}

message BotAction {
    int32 action_type = 1;  // 0-7 arası aksiyon indeksi
    repeated float movement_direction = 2;  // Normalized direction
    repeated float aim_direction = 3;
    bool should_fire = 4;
    bool should_reload = 5;
}

message Experience {
    GameState state = 1;
    BotAction action = 2;
    float reward = 3;
    GameState next_state = 4;
    bool done = 5;
}

message Acknowledgment {
    bool success = 1;
    string message = 2;
}

message TrainingCommand {
    enum Command {
        START = 0;
        STOP = 1;
        SAVE = 2;
        LOAD = 3;
    }
    Command command = 1;
    string model_path = 2;
}

message TrainingStatus {
    bool is_training = 1;
    int32 total_steps = 2;
    float average_reward = 3;
    float loss = 4;
}
```

#### 6.1.2 Python RL Server

```python
# rl_server/bot_rl_server.py
import grpc
from concurrent import futures
import numpy as np
import torch
import torch.nn as nn
from stable_baselines3 import PPO
from stable_baselines3.common.vec_env import DummyVecEnv
import gymnasium as gym
from gymnasium import spaces
import threading
import queue
import logging

# Proto generated files
import bot_ai_pb2
import bot_ai_pb2_grpc

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class BotEnvironment(gym.Env):
    """Unreal Engine ile iletişim kuran Gymnasium environment."""
    
    def __init__(self):
        super().__init__()
        
        # Observation space: 64 boyutlu vektör
        # [bot_health, bot_ammo, bot_pos(3), bot_rot(3), 
        #  has_target, target_pos(3), target_health, distance, los,
        #  cover_positions(30), num_enemies, num_allies, game_time, phase]
        self.observation_space = spaces.Box(
            low=-np.inf, high=np.inf, shape=(64,), dtype=np.float32
        )
        
        # Action space: 8 ayrık aksiyon
        # 0: Attack, 1: TakeCover, 2: Flee, 3: Reload
        # 4: Patrol, 5: Investigate, 6: Support, 7: Flank
        self.action_space = spaces.Discrete(8)
        
        # State queue (UE'den gelen state'ler)
        self.state_queue = queue.Queue()
        self.current_state = None
        self.last_reward = 0.0
        self.episode_done = False
        
    def _game_state_to_observation(self, game_state: bot_ai_pb2.GameState) -> np.ndarray:
        """GameState proto'yu numpy observation'a çevir."""
        obs = np.zeros(64, dtype=np.float32)
        
        # Bot durumu
        obs[0] = game_state.bot_health / 100.0  # Normalize
        obs[1] = game_state.bot_ammo / 30.0  # Normalize (max 30 mermi varsayımı)
        obs[2:5] = np.array(game_state.bot_position) / 10000.0  # Scale down
        obs[5:8] = np.array(game_state.bot_rotation) / 180.0  # Normalize to [-1, 1]
        
        # Hedef durumu
        obs[8] = 1.0 if game_state.has_target else 0.0
        if game_state.has_target:
            obs[9:12] = np.array(game_state.target_position) / 10000.0
            obs[12] = game_state.target_health / 100.0
            obs[13] = min(game_state.distance_to_target / 2000.0, 1.0)
            obs[14] = 1.0 if game_state.has_line_of_sight else 0.0
        
        # Cover pozisyonları (max 10 cover, her biri 3 koordinat)
        cover_data = list(game_state.nearby_cover_positions)[:30]
        obs[15:15+len(cover_data)] = np.array(cover_data) / 10000.0
        
        # Diğer bilgiler
        obs[45] = min(game_state.num_nearby_enemies / 5.0, 1.0)
        obs[46] = min(game_state.num_nearby_allies / 5.0, 1.0)
        obs[47] = game_state.game_time / 600.0  # 10 dakika normalize
        obs[48] = game_state.current_phase / 2.0
        
        return obs
    
    def step(self, action):
        """Environment step - UE'den yeni state bekle."""
        # Bu fonksiyon aslında UE tarafından tetiklenir
        # Burada sadece placeholder
        
        if self.current_state is None:
            obs = np.zeros(64, dtype=np.float32)
        else:
            obs = self._game_state_to_observation(self.current_state)
        
        return obs, self.last_reward, self.episode_done, False, {}
    
    def reset(self, seed=None):
        """Environment reset."""
        super().reset(seed=seed)
        self.episode_done = False
        self.last_reward = 0.0
        
        if self.current_state is None:
            obs = np.zeros(64, dtype=np.float32)
        else:
            obs = self._game_state_to_observation(self.current_state)
            
        return obs, {}
    
    def update_state(self, game_state: bot_ai_pb2.GameState):
        """UE'den gelen state'i güncelle."""
        self.current_state = game_state
    
    def set_reward(self, reward: float, done: bool):
        """Reward ve done flag'i ayarla."""
        self.last_reward = reward
        self.episode_done = done


class RewardCalculator:
    """Reward hesaplama sınıfı."""
    
    def __init__(self):
        self.previous_health = 100.0
        self.previous_target_health = 100.0
        self.kill_count = 0
        self.death_count = 0
        
    def calculate_reward(self, state: bot_ai_pb2.GameState, 
                         action: int, next_state: bot_ai_pb2.GameState) -> float:
        """Detaylı reward hesaplama."""
        reward = 0.0
        
        # Hayatta kalma ödülü
        reward += 0.01
        
        # Verilen hasar ödülü
        if state.has_target and next_state.has_target:
            damage_dealt = state.target_health - next_state.target_health
            if damage_dealt > 0:
                reward += damage_dealt * 0.1
        
        # Öldürme ödülü
        if state.has_target and state.target_health > 0:
            if not next_state.has_target or next_state.target_health <= 0:
                reward += 10.0
                self.kill_count += 1
        
        # Alınan hasar cezası
        damage_taken = state.bot_health - next_state.bot_health
        if damage_taken > 0:
            reward -= damage_taken * 0.05
        
        # Ölüm cezası
        if next_state.bot_health <= 0:
            reward -= 20.0
            self.death_count += 1
        
        # Aksiyon bazlı ödüller
        if action == 0:  # Attack
            if state.has_line_of_sight and state.has_target:
                reward += 0.1  # Doğru zamanda saldırı
            else:
                reward -= 0.05  # Yanlış zamanda saldırı
                
        elif action == 1:  # TakeCover
            if state.bot_health < 50 or not state.has_line_of_sight:
                reward += 0.1  # Doğru zamanda siper
                
        elif action == 2:  # Flee
            if state.bot_health < 20:
                reward += 0.2  # Kritik durumda kaçış
            else:
                reward -= 0.1  # Gereksiz kaçış
        
        # Mesafe bazlı ödül (hedef varsa)
        if state.has_target:
            # Optimal mesafe: 500-1000 birim
            distance = state.distance_to_target
            if 500 <= distance <= 1000:
                reward += 0.02
            elif distance < 300:
                reward -= 0.01  # Çok yakın
            elif distance > 1500:
                reward -= 0.01  # Çok uzak
        
        return reward
    
    def reset(self):
        """Yeni episode için reset."""
        self.previous_health = 100.0
        self.previous_target_health = 100.0


class BotAIServicer(bot_ai_pb2_grpc.BotAIServiceServicer):
    """gRPC service implementation."""
    
    def __init__(self):
        self.env = BotEnvironment()
        self.model = None
        self.is_training = False
        self.experience_buffer = []
        self.reward_calculator = RewardCalculator()
        self.training_thread = None
        self.total_steps = 0
        self.average_reward = 0.0
        
        # Model yükle veya oluştur
        self._initialize_model()
        
    def _initialize_model(self):
        """PPO modelini oluştur veya yükle."""
        try:
            self.model = PPO.load("models/bot_policy")
            logger.info("Mevcut model yüklendi.")
        except:
            logger.info("Yeni model oluşturuluyor...")
            vec_env = DummyVecEnv([lambda: self.env])
            self.model = PPO(
                "MlpPolicy",
                vec_env,
                verbose=1,
                learning_rate=3e-4,
                n_steps=2048,
                batch_size=64,
                n_epochs=10,
                gamma=0.99,
                gae_lambda=0.95,
                clip_range=0.2,
                tensorboard_log="./logs/"
            )
    
    def GetAction(self, request: bot_ai_pb2.GameState, 
                  context) -> bot_ai_pb2.BotAction:
        """Mevcut state için aksiyon döndür."""
        
        # State'i environment'a güncelle
        self.env.update_state(request)
        
        # Observation oluştur
        obs = self.env._game_state_to_observation(request)
        
        # Model'den aksiyon al
        if self.model:
            action, _ = self.model.predict(obs, deterministic=not self.is_training)
        else:
            action = np.random.randint(0, 8)
        
        # BotAction response oluştur
        response = bot_ai_pb2.BotAction()
        response.action_type = int(action)
        
        # Aksiyon tipine göre ek bilgiler
        if action == 0:  # Attack
            response.should_fire = True
            if request.has_target:
                # Hedefe doğru aim
                target_dir = np.array(request.target_position) - np.array(request.bot_position)
                target_dir = target_dir / (np.linalg.norm(target_dir) + 1e-8)
                response.aim_direction.extend(target_dir.tolist())
        elif action == 3:  # Reload
            response.should_reload = True
        
        return response
    
    def SendExperience(self, request: bot_ai_pb2.Experience, 
                       context) -> bot_ai_pb2.Acknowledgment:
        """Eğitim için deneyim al."""
        
        # Reward hesapla
        reward = self.reward_calculator.calculate_reward(
            request.state, 
            request.action.action_type,
            request.next_state
        )
        
        # Experience buffer'a ekle
        self.experience_buffer.append({
            'state': self.env._game_state_to_observation(request.state),
            'action': request.action.action_type,
            'reward': reward,
            'next_state': self.env._game_state_to_observation(request.next_state),
            'done': request.done
        })
        
        # Buffer dolunca eğitim yap
        if len(self.experience_buffer) >= 2048 and self.is_training:
            self._train_on_buffer()
        
        return bot_ai_pb2.Acknowledgment(success=True, message="Experience received")
    
    def _train_on_buffer(self):
        """Buffer'daki deneyimlerle eğitim yap."""
        if not self.model or len(self.experience_buffer) == 0:
            return
            
        logger.info(f"Training on {len(self.experience_buffer)} experiences...")
        
        # Bu basitleştirilmiş bir eğitim döngüsü
        # Gerçek implementasyonda daha sofistike olmalı
        
        # Average reward hesapla
        rewards = [exp['reward'] for exp in self.experience_buffer]
        self.average_reward = np.mean(rewards)
        self.total_steps += len(self.experience_buffer)
        
        logger.info(f"Average reward: {self.average_reward:.3f}, Total steps: {self.total_steps}")
        
        # Buffer'ı temizle
        self.experience_buffer = []
    
    def ControlTraining(self, request: bot_ai_pb2.TrainingCommand, 
                        context) -> bot_ai_pb2.TrainingStatus:
        """Eğitim kontrolü."""
        
        if request.command == bot_ai_pb2.TrainingCommand.START:
            self.is_training = True
            logger.info("Training started")
            
        elif request.command == bot_ai_pb2.TrainingCommand.STOP:
            self.is_training = False
            logger.info("Training stopped")
            
        elif request.command == bot_ai_pb2.TrainingCommand.SAVE:
            if self.model:
                path = request.model_path or "models/bot_policy"
                self.model.save(path)
                logger.info(f"Model saved to {path}")
                
        elif request.command == bot_ai_pb2.TrainingCommand.LOAD:
            if request.model_path:
                self.model = PPO.load(request.model_path)
                logger.info(f"Model loaded from {request.model_path}")
        
        return bot_ai_pb2.TrainingStatus(
            is_training=self.is_training,
            total_steps=self.total_steps,
            average_reward=self.average_reward,
            loss=0.0  # Placeholder
        )


def serve():
    """gRPC server'ı başlat."""
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    bot_ai_pb2_grpc.add_BotAIServiceServicer_to_server(BotAIServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    logger.info("Bot AI RL Server started on port 50051")
    server.wait_for_termination()


if __name__ == '__main__':
    serve()
```

### 6.2 Unreal Engine gRPC Client

```cpp
// Source/BotAI/Public/AI/RL/RLBridgeComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "RLBridgeComponent.generated.h"

USTRUCT(BlueprintType)
struct FRLGameState
{
    GENERATED_BODY()

    UPROPERTY() float BotHealth;
    UPROPERTY() float BotAmmo;
    UPROPERTY() FVector BotPosition;
    UPROPERTY() FRotator BotRotation;
    UPROPERTY() bool bHasTarget;
    UPROPERTY() FVector TargetPosition;
    UPROPERTY() float TargetHealth;
    UPROPERTY() float DistanceToTarget;
    UPROPERTY() bool bHasLineOfSight;
    UPROPERTY() TArray<FVector> NearbyCoverPositions;
    UPROPERTY() int32 NumNearbyEnemies;
    UPROPERTY() int32 NumNearbyAllies;
    UPROPERTY() float GameTime;
    UPROPERTY() int32 CurrentPhase;
};

USTRUCT(BlueprintType)
struct FRLAction
{
    GENERATED_BODY()

    UPROPERTY() int32 ActionType;
    UPROPERTY() FVector MovementDirection;
    UPROPERTY() FVector AimDirection;
    UPROPERTY() bool bShouldFire;
    UPROPERTY() bool bShouldReload;
};

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class BOTAI_API URLBridgeComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    URLBridgeComponent();

    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
        FActorComponentTickFunction* ThisTickFunction) override;

    // Ana fonksiyonlar
    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    FRLAction GetActionFromRL(const FRLGameState& GameState);

    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    void SendExperience(const FRLGameState& State, const FRLAction& Action, 
        float Reward, const FRLGameState& NextState, bool bDone);

    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    void StartTraining();

    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    void StopTraining();

    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    void SaveModel(const FString& Path);

    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    void LoadModel(const FString& Path);

    // Bağlantı durumu
    UFUNCTION(BlueprintCallable, Category = "RL Bridge")
    bool IsConnected() const { return bIsConnected; }

protected:
    UPROPERTY(EditAnywhere, Category = "RL Bridge")
    FString ServerAddress = "localhost:50051";

    UPROPERTY(EditAnywhere, Category = "RL Bridge")
    float ConnectionTimeout = 5.0f;

private:
    bool bIsConnected = false;
    
    // TCP/gRPC bağlantısı için implementation
    // Not: Gerçek implementasyonda gRPC veya custom TCP kullanılmalı
    void ConnectToServer();
    void DisconnectFromServer();
    
    // JSON serialization (basit implementasyon için)
    FString SerializeGameState(const FRLGameState& State);
    FRLAction DeserializeAction(const FString& JsonString);
};
```

---

## 7. Faz 5: Adaptif Zorluk Sistemi

### 7.1 Oyuncu Performans Takibi

```cpp
// Source/BotAI/Public/AI/DDA/PlayerPerformanceTracker.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "PlayerPerformanceTracker.generated.h"

USTRUCT(BlueprintType)
struct FPlayerPerformanceMetrics
{
    GENERATED_BODY()

    // Doğruluk metrikleri
    UPROPERTY(BlueprintReadOnly) float AccuracyRatio = 0.5f;
    UPROPERTY(BlueprintReadOnly) int32 ShotsFired = 0;
    UPROPERTY(BlueprintReadOnly) int32 ShotsHit = 0;
    
    // Tepki metrikleri
    UPROPERTY(BlueprintReadOnly) float AverageReactionTime = 500.0f;  // ms
    UPROPERTY(BlueprintReadOnly) TArray<float> ReactionTimeHistory;
    
    // Hayatta kalma metrikleri
    UPROPERTY(BlueprintReadOnly) float AverageLifespan = 60.0f;  // seconds
    UPROPERTY(BlueprintReadOnly) TArray<float> LifespanHistory;
    
    // Savaş metrikleri
    UPROPERTY(BlueprintReadOnly) float KillDeathRatio = 1.0f;
    UPROPERTY(BlueprintReadOnly) int32 TotalKills = 0;
    UPROPERTY(BlueprintReadOnly) int32 TotalDeaths = 0;
    
    // Görev metrikleri
    UPROPERTY(BlueprintReadOnly) float ObjectiveCompletionRate = 0.5f;
    UPROPERTY(BlueprintReadOnly) int32 ObjectivesCompleted = 0;
    UPROPERTY(BlueprintReadOnly) int32 ObjectivesAttempted = 0;
    
    // Hasar metrikleri
    UPROPERTY(BlueprintReadOnly) float DamageTakenPerMinute = 0.0f;
    UPROPERTY(BlueprintReadOnly) float DamageDealtPerMinute = 0.0f;
    UPROPERTY(BlueprintReadOnly) float TotalDamageTaken = 0.0f;
    UPROPERTY(BlueprintReadOnly) float TotalDamageDealt = 0.0f;
    
    // Zaman metrikleri
    UPROPERTY(BlueprintReadOnly) float TotalPlayTime = 0.0f;
    UPROPERTY(BlueprintReadOnly) float CurrentSessionTime = 0.0f;
    
    // Hesaplanmış skill skoru (0-1 arası)
    UPROPERTY(BlueprintReadOnly) float OverallSkillScore = 0.5f;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPerformanceUpdated, 
    const FPlayerPerformanceMetrics&, Metrics);

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class BOTAI_API UPlayerPerformanceTracker : public UActorComponent
{
    GENERATED_BODY()

public:
    UPlayerPerformanceTracker();

    virtual void BeginPlay() override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
        FActorComponentTickFunction* ThisTickFunction) override;

    // Event handlers
    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnPlayerFiredShot(bool bHit);

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnPlayerKill();

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnPlayerDeath();

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnPlayerDamageDealt(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnPlayerDamageTaken(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnObjectiveCompleted();

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnObjectiveFailed();

    UFUNCTION(BlueprintCallable, Category = "Performance")
    void OnEnemySpotted(float ReactionTimeMs);

    // Getters
    UFUNCTION(BlueprintCallable, Category = "Performance")
    FPlayerPerformanceMetrics GetMetrics() const { return Metrics; }

    UFUNCTION(BlueprintCallable, Category = "Performance")
    float GetSkillScore() const { return Metrics.OverallSkillScore; }

    // Delegate
    UPROPERTY(BlueprintAssignable, Category = "Performance")
    FOnPerformanceUpdated OnPerformanceUpdated;

protected:
    UPROPERTY(EditAnywhere, Category = "Performance")
    float UpdateInterval = 1.0f;

    UPROPERTY(EditAnywhere, Category = "Performance")
    int32 HistorySize = 20;

    // Skill score ağırlıkları
    UPROPERTY(EditAnywhere, Category = "Performance|Weights")
    float AccuracyWeight = 0.25f;

    UPROPERTY(EditAnywhere, Category = "Performance|Weights")
    float ReactionTimeWeight = 0.20f;

    UPROPERTY(EditAnywhere, Category = "Performance|Weights")
    float KDRatioWeight = 0.30f;

    UPROPERTY(EditAnywhere, Category = "Performance|Weights")
    float ObjectiveWeight = 0.25f;

private:
    FPlayerPerformanceMetrics Metrics;
    float TimeSinceLastUpdate = 0.0f;
    float CurrentLifeStartTime = 0.0f;

    void UpdateMetrics();
    void CalculateSkillScore();
    void AddToHistory(TArray<float>& History, float Value);
    float GetAverageFromHistory(const TArray<float>& History) const;
};
```

```cpp
// Source/BotAI/Private/AI/DDA/PlayerPerformanceTracker.cpp
#include "AI/DDA/PlayerPerformanceTracker.h"

UPlayerPerformanceTracker::UPlayerPerformanceTracker()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UPlayerPerformanceTracker::BeginPlay()
{
    Super::BeginPlay();
    CurrentLifeStartTime = GetWorld()->GetTimeSeconds();
}

void UPlayerPerformanceTracker::TickComponent(float DeltaTime, ELevelTick TickType, 
    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    Metrics.CurrentSessionTime += DeltaTime;
    Metrics.TotalPlayTime += DeltaTime;

    TimeSinceLastUpdate += DeltaTime;
    if (TimeSinceLastUpdate >= UpdateInterval)
    {
        UpdateMetrics();
        TimeSinceLastUpdate = 0.0f;
    }
}

void UPlayerPerformanceTracker::OnPlayerFiredShot(bool bHit)
{
    Metrics.ShotsFired++;
    if (bHit)
    {
        Metrics.ShotsHit++;
    }
    
    if (Metrics.ShotsFired > 0)
    {
        Metrics.AccuracyRatio = static_cast<float>(Metrics.ShotsHit) / 
            static_cast<float>(Metrics.ShotsFired);
    }
}

void UPlayerPerformanceTracker::OnPlayerKill()
{
    Metrics.TotalKills++;
    
    if (Metrics.TotalDeaths > 0)
    {
        Metrics.KillDeathRatio = static_cast<float>(Metrics.TotalKills) / 
            static_cast<float>(Metrics.TotalDeaths);
    }
    else
    {
        Metrics.KillDeathRatio = static_cast<float>(Metrics.TotalKills);
    }
}

void UPlayerPerformanceTracker::OnPlayerDeath()
{
    Metrics.TotalDeaths++;
    
    // Lifespan hesapla
    float CurrentLifespan = GetWorld()->GetTimeSeconds() - CurrentLifeStartTime;
    AddToHistory(Metrics.LifespanHistory, CurrentLifespan);
    Metrics.AverageLifespan = GetAverageFromHistory(Metrics.LifespanHistory);
    
    // K/D güncelle
    Metrics.KillDeathRatio = static_cast<float>(Metrics.TotalKills) / 
        FMath::Max(1.0f, static_cast<float>(Metrics.TotalDeaths));
    
    // Yeni hayat başlat
    CurrentLifeStartTime = GetWorld()->GetTimeSeconds();
}

void UPlayerPerformanceTracker::OnPlayerDamageDealt(float Amount)
{
    Metrics.TotalDamageDealt += Amount;
}

void UPlayerPerformanceTracker::OnPlayerDamageTaken(float Amount)
{
    Metrics.TotalDamageTaken += Amount;
}

void UPlayerPerformanceTracker::OnObjectiveCompleted()
{
    Metrics.ObjectivesCompleted++;
    Metrics.ObjectivesAttempted++;
    
    if (Metrics.ObjectivesAttempted > 0)
    {
        Metrics.ObjectiveCompletionRate = static_cast<float>(Metrics.ObjectivesCompleted) / 
            static_cast<float>(Metrics.ObjectivesAttempted);
    }
}

void UPlayerPerformanceTracker::OnObjectiveFailed()
{
    Metrics.ObjectivesAttempted++;
    
    if (Metrics.ObjectivesAttempted > 0)
    {
        Metrics.ObjectiveCompletionRate = static_cast<float>(Metrics.ObjectivesCompleted) / 
            static_cast<float>(Metrics.ObjectivesAttempted);
    }
}

void UPlayerPerformanceTracker::OnEnemySpotted(float ReactionTimeMs)
{
    AddToHistory(Metrics.ReactionTimeHistory, ReactionTimeMs);
    Metrics.AverageReactionTime = GetAverageFromHistory(Metrics.ReactionTimeHistory);
}

void UPlayerPerformanceTracker::UpdateMetrics()
{
    // DPM hesapla
    if (Metrics.TotalPlayTime > 0)
    {
        float Minutes = Metrics.TotalPlayTime / 60.0f;
        Metrics.DamageTakenPerMinute = Metrics.TotalDamageTaken / Minutes;
        Metrics.DamageDealtPerMinute = Metrics.TotalDamageDealt / Minutes;
    }

    CalculateSkillScore();
    OnPerformanceUpdated.Broadcast(Metrics);
}

void UPlayerPerformanceTracker::CalculateSkillScore()
{
    // Accuracy score (0-1)
    float AccuracyScore = Metrics.AccuracyRatio;

    // Reaction time score (hızlı = yüksek)
    // 200ms = 1.0, 1000ms = 0.0
    float ReactionScore = FMath::Clamp(
        1.0f - (Metrics.AverageReactionTime - 200.0f) / 800.0f,
        0.0f, 1.0f
    );

    // K/D score
    // K/D 3.0+ = 1.0, K/D 0.0 = 0.0
    float KDScore = FMath::Clamp(Metrics.KillDeathRatio / 3.0f, 0.0f, 1.0f);

    // Objective score
    float ObjectiveScore = Metrics.ObjectiveCompletionRate;

    // Ağırlıklı ortalama
    Metrics.OverallSkillScore = 
        AccuracyScore * AccuracyWeight +
        ReactionScore * ReactionTimeWeight +
        KDScore * KDRatioWeight +
        ObjectiveScore * ObjectiveWeight;

    Metrics.OverallSkillScore = FMath::Clamp(Metrics.OverallSkillScore, 0.0f, 1.0f);
}

void UPlayerPerformanceTracker::AddToHistory(TArray<float>& History, float Value)
{
    History.Add(Value);
    
    while (History.Num() > HistorySize)
    {
        History.RemoveAt(0);
    }
}

float UPlayerPerformanceTracker::GetAverageFromHistory(const TArray<float>& History) const
{
    if (History.Num() == 0) return 0.0f;
    
    float Sum = 0.0f;
    for (float Value : History)
    {
        Sum += Value;
    }
    return Sum / static_cast<float>(History.Num());
}
```

### 7.2 Difficulty Manager

```cpp
// Source/BotAI/Public/AI/DDA/DifficultyManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "AI/DDA/PlayerPerformanceTracker.h"
#include "DifficultyManager.generated.h"

USTRUCT(BlueprintType)
struct FBotDifficultyParams
{
    GENERATED_BODY()

    // Algılama parametreleri
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float SightRange = 2000.0f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float HearingRange = 1500.0f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float ReactionDelay = 0.3f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float PeripheralVision = 60.0f;

    // Savaş parametreleri
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float AimAccuracy = 0.7f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float AimSpeed = 5.0f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float BurstDuration = 0.5f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float FireRateMultiplier = 1.0f;

    // Taktik parametreleri
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float AggressionLevel = 0.5f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float FlankingProbability = 0.3f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float CoverUsageFrequency = 0.5f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float TeamCoordinationLevel = 0.5f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float RetreatThreshold = 0.3f;

    // Hareket parametreleri
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float MovementSpeedMultiplier = 1.0f;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) float DodgeProbability = 0.2f;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDifficultyChanged, 
    float, OldDifficulty, float, NewDifficulty);

UCLASS()
class BOTAI_API UDifficultyManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // Ana zorluk kontrolü
    UFUNCTION(BlueprintCallable, Category = "DDA")
    void UpdateDifficulty(const FPlayerPerformanceMetrics& PlayerMetrics);

    UFUNCTION(BlueprintCallable, Category = "DDA")
    float GetCurrentDifficulty() const { return CurrentDifficulty; }

    UFUNCTION(BlueprintCallable, Category = "DDA")
    FBotDifficultyParams GetBotParams() const { return CurrentBotParams; }

    // Manuel zorluk ayarı
    UFUNCTION(BlueprintCallable, Category = "DDA")
    void SetBaseDifficulty(float NewBaseDifficulty);

    UFUNCTION(BlueprintCallable, Category = "DDA")
    void SetAdaptiveEnabled(bool bEnabled) { bAdaptiveEnabled = bEnabled; }

    // Delegate
    UPROPERTY(BlueprintAssignable, Category = "DDA")
    FOnDifficultyChanged OnDifficultyChanged;

protected:
    // Zorluk aralığı
    UPROPERTY(EditAnywhere, Category = "DDA")
    float MinDifficulty = 0.1f;

    UPROPERTY(EditAnywhere, Category = "DDA")
    float MaxDifficulty = 1.0f;

    // Değişim hızı
    UPROPERTY(EditAnywhere, Category = "DDA")
    float DifficultyChangeSpeed = 0.5f;

    // Adaptif zorluk aktif mi?
    UPROPERTY(EditAnywhere, Category = "DDA")
    bool bAdaptiveEnabled = true;

    // Hedef performans
    UPROPERTY(EditAnywhere, Category = "DDA")
    float TargetWinRate = 0.5f;

private:
    float CurrentDifficulty = 0.5f;
    float BaseDifficulty = 0.5f;
    FBotDifficultyParams CurrentBotParams;

    float CalculateTargetDifficulty(const FPlayerPerformanceMetrics& Metrics);
    void UpdateBotParameters(float Difficulty);
    float LerpDifficulty(float Current, float Target, float DeltaTime);
};
```

```cpp
// Source/BotAI/Private/AI/DDA/DifficultyManager.cpp
#include "AI/DDA/DifficultyManager.h"

void UDifficultyManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    CurrentDifficulty = BaseDifficulty;
    UpdateBotParameters(CurrentDifficulty);
}

void UDifficultyManager::UpdateDifficulty(const FPlayerPerformanceMetrics& PlayerMetrics)
{
    if (!bAdaptiveEnabled) return;

    float TargetDifficulty = CalculateTargetDifficulty(PlayerMetrics);
    float OldDifficulty = CurrentDifficulty;

    // Yumuşak geçiş
    CurrentDifficulty = FMath::FInterpTo(
        CurrentDifficulty,
        TargetDifficulty,
        GetWorld()->GetDeltaSeconds(),
        DifficultyChangeSpeed
    );

    CurrentDifficulty = FMath::Clamp(CurrentDifficulty, MinDifficulty, MaxDifficulty);

    if (!FMath::IsNearlyEqual(OldDifficulty, CurrentDifficulty, 0.01f))
    {
        UpdateBotParameters(CurrentDifficulty);
        OnDifficultyChanged.Broadcast(OldDifficulty, CurrentDifficulty);
    }
}

float UDifficultyManager::CalculateTargetDifficulty(const FPlayerPerformanceMetrics& Metrics)
{
    // Oyuncu skill score'una göre zorluk ayarla
    // Yüksek skill = yüksek zorluk
    float SkillBasedDifficulty = Metrics.OverallSkillScore;

    // K/D oranına göre ayarlama
    // K/D > 2 ise zorlaştır, K/D < 0.5 ise kolaylaştır
    float KDModifier = 0.0f;
    if (Metrics.KillDeathRatio > 2.0f)
    {
        KDModifier = FMath::Min((Metrics.KillDeathRatio - 2.0f) * 0.1f, 0.2f);
    }
    else if (Metrics.KillDeathRatio < 0.5f)
    {
        KDModifier = -FMath::Min((0.5f - Metrics.KillDeathRatio) * 0.2f, 0.2f);
    }

    // Son ölümlerden sonra geçici kolaylaştırma
    float RecentDeathModifier = 0.0f;
    if (Metrics.AverageLifespan < 30.0f)  // 30 saniyeden az hayatta kalıyorsa
    {
        RecentDeathModifier = -0.1f;
    }

    float TargetDifficulty = BaseDifficulty + SkillBasedDifficulty * 0.3f + 
        KDModifier + RecentDeathModifier;

    return FMath::Clamp(TargetDifficulty, MinDifficulty, MaxDifficulty);
}

void UDifficultyManager::UpdateBotParameters(float Difficulty)
{
    // Difficulty 0.0-1.0 arası
    // Parametreleri lineer interpolasyon ile ayarla

    // Algılama
    CurrentBotParams.SightRange = FMath::Lerp(1000.0f, 3000.0f, Difficulty);
    CurrentBotParams.HearingRange = FMath::Lerp(800.0f, 2000.0f, Difficulty);
    CurrentBotParams.ReactionDelay = FMath::Lerp(0.8f, 0.1f, Difficulty);
    CurrentBotParams.PeripheralVision = FMath::Lerp(45.0f, 90.0f, Difficulty);

    // Savaş
    CurrentBotParams.AimAccuracy = FMath::Lerp(0.3f, 0.95f, Difficulty);
    CurrentBotParams.AimSpeed = FMath::Lerp(2.0f, 10.0f, Difficulty);
    CurrentBotParams.BurstDuration = FMath::Lerp(0.2f, 1.0f, Difficulty);
    CurrentBotParams.FireRateMultiplier = FMath::Lerp(0.5f, 1.5f, Difficulty);

    // Taktik
    CurrentBotParams.AggressionLevel = FMath::Lerp(0.2f, 0.9f, Difficulty);
    CurrentBotParams.FlankingProbability = FMath::Lerp(0.1f, 0.6f, Difficulty);
    CurrentBotParams.CoverUsageFrequency = FMath::Lerp(0.3f, 0.8f, Difficulty);
    CurrentBotParams.TeamCoordinationLevel = FMath::Lerp(0.2f, 0.9f, Difficulty);
    CurrentBotParams.RetreatThreshold = FMath::Lerp(0.5f, 0.2f, Difficulty);

    // Hareket
    CurrentBotParams.MovementSpeedMultiplier = FMath::Lerp(0.7f, 1.3f, Difficulty);
    CurrentBotParams.DodgeProbability = FMath::Lerp(0.05f, 0.4f, Difficulty);
}

void UDifficultyManager::SetBaseDifficulty(float NewBaseDifficulty)
{
    BaseDifficulty = FMath::Clamp(NewBaseDifficulty, MinDifficulty, MaxDifficulty);
    
    if (!bAdaptiveEnabled)
    {
        CurrentDifficulty = BaseDifficulty;
        UpdateBotParameters(CurrentDifficulty);
    }
}
```

---

## 8. Faz 6: Inverse Kinematics (IK) Sistemi

### 8.1 Control Rig Setup Rehberi

Unreal Engine'de Control Rig kullanarak IK sistemi kurmak için aşağıdaki adımları izleyin:

#### 8.1.1 Control Rig Oluşturma

1. Content Browser'da sağ tık > Animation > Control Rig
2. İsim: `CR_BotCharacter`
3. Skeleton seçin: BotCharacter'ın skeleton'ı

#### 8.1.2 Foot IK Setup

```cpp
// Control Rig içinde (Blueprint/Rig Graph)
// Foot IK için iki ana kontrol noktası oluşturun:
// - IK_Foot_L (Sol ayak)
// - IK_Foot_R (Sağ ayak)

// İşlem sırası:
// 1. Trace - Ayak altına ray cast
// 2. Adjust - Pozisyon ayarla
// 3. Rotate - Zemin normaline göre döndür
```

### 8.2 Animation Blueprint Integration

```cpp
// Source/BotAI/Public/Animation/BotAnimInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "BotAnimInstance.generated.h"

UCLASS()
class BOTAI_API UBotAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

    // Foot IK
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Foot")
    FVector LeftFootOffset;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Foot")
    FVector RightFootOffset;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Foot")
    FRotator LeftFootRotation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Foot")
    FRotator RightFootRotation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Foot")
    float PelvisOffset;

    // Aim IK
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Aim")
    FRotator AimRotation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|Aim")
    FVector AimTarget;

    // Look At IK
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|LookAt")
    FVector LookAtTarget;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "IK|LookAt")
    float LookAtWeight;

    // Movement
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    float Speed;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    bool bIsInAir;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
    bool bIsCrouching;

protected:
    UPROPERTY()
    class ABotCharacter* BotCharacter;

    // Foot IK trace
    UPROPERTY(EditAnywhere, Category = "IK|Foot")
    float FootTraceDistance = 50.0f;

    UPROPERTY(EditAnywhere, Category = "IK|Foot")
    float IKInterpSpeed = 15.0f;

private:
    void UpdateFootIK(float DeltaTime);
    void UpdateAimIK();
    void UpdateLookAt();
    
    FVector TraceFootPosition(FName SocketName);
    FRotator CalculateFootRotation(const FHitResult& HitResult);
};
```

```cpp
// Source/BotAI/Private/Animation/BotAnimInstance.cpp
#include "Animation/BotAnimInstance.h"
#include "Characters/BotCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

void UBotAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();

    APawn* Pawn = TryGetPawnOwner();
    if (Pawn)
    {
        BotCharacter = Cast<ABotCharacter>(Pawn);
    }
}

void UBotAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    if (!BotCharacter) return;

    // Movement variables
    UCharacterMovementComponent* Movement = BotCharacter->GetCharacterMovement();
    if (Movement)
    {
        Speed = Movement->Velocity.Size();
        bIsInAir = Movement->IsFalling();
        bIsCrouching = Movement->IsCrouching();
    }

    // IK Updates
    UpdateFootIK(DeltaSeconds);
    UpdateAimIK();
    UpdateLookAt();
}

void UBotAnimInstance::UpdateFootIK(float DeltaTime)
{
    if (!BotCharacter || bIsInAir) 
    {
        // Havadayken IK'yı sıfırla
        LeftFootOffset = FMath::VInterpTo(LeftFootOffset, FVector::ZeroVector, 
            DeltaTime, IKInterpSpeed);
        RightFootOffset = FMath::VInterpTo(RightFootOffset, FVector::ZeroVector, 
            DeltaTime, IKInterpSpeed);
        PelvisOffset = FMath::FInterpTo(PelvisOffset, 0.0f, DeltaTime, IKInterpSpeed);
        return;
    }

    // Sol ayak trace
    FVector LeftFootTraceResult = TraceFootPosition(FName("foot_l"));
    FVector TargetLeftOffset = LeftFootTraceResult - BotCharacter->GetMesh()->
        GetSocketLocation(FName("foot_l"));
    TargetLeftOffset.Z = FMath::Clamp(TargetLeftOffset.Z, -FootTraceDistance, 
        FootTraceDistance);
    
    // Sağ ayak trace
    FVector RightFootTraceResult = TraceFootPosition(FName("foot_r"));
    FVector TargetRightOffset = RightFootTraceResult - BotCharacter->GetMesh()->
        GetSocketLocation(FName("foot_r"));
    TargetRightOffset.Z = FMath::Clamp(TargetRightOffset.Z, -FootTraceDistance, 
        FootTraceDistance);

    // Smooth interpolation
    LeftFootOffset = FMath::VInterpTo(LeftFootOffset, TargetLeftOffset, 
        DeltaTime, IKInterpSpeed);
    RightFootOffset = FMath::VInterpTo(RightFootOffset, TargetRightOffset, 
        DeltaTime, IKInterpSpeed);

    // Pelvis offset (daha düşük ayağa göre)
    float TargetPelvisOffset = FMath::Min(LeftFootOffset.Z, RightFootOffset.Z);
    PelvisOffset = FMath::FInterpTo(PelvisOffset, TargetPelvisOffset, 
        DeltaTime, IKInterpSpeed);
}

FVector UBotAnimInstance::TraceFootPosition(FName SocketName)
{
    if (!BotCharacter) return FVector::ZeroVector;

    FVector SocketLocation = BotCharacter->GetMesh()->GetSocketLocation(SocketName);
    FVector StartLocation = SocketLocation + FVector(0, 0, FootTraceDistance);
    FVector EndLocation = SocketLocation - FVector(0, 0, FootTraceDistance * 2);

    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(BotCharacter);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        ECC_Visibility,
        QueryParams
    );

    if (bHit)
    {
        // Ayak rotasyonunu da hesapla
        if (SocketName == FName("foot_l"))
        {
            LeftFootRotation = CalculateFootRotation(HitResult);
        }
        else
        {
            RightFootRotation = CalculateFootRotation(HitResult);
        }
        
        return HitResult.Location;
    }

    return SocketLocation;
}

FRotator UBotAnimInstance::CalculateFootRotation(const FHitResult& HitResult)
{
    FVector ImpactNormal = HitResult.ImpactNormal;
    
    // Zemin normalinden rotasyon hesapla
    FRotator TargetRotation = UKismetMathLibrary::MakeRotFromZX(
        ImpactNormal, 
        BotCharacter->GetActorForwardVector()
    );

    return TargetRotation;
}

void UBotAnimInstance::UpdateAimIK()
{
    if (!BotCharacter) return;

    // AI Controller'dan hedef bilgisi al
    AAIController* AIController = Cast<AAIController>(BotCharacter->GetController());
    if (AIController)
    {
        AActor* FocusActor = AIController->GetFocusActor();
        if (FocusActor)
        {
            AimTarget = FocusActor->GetActorLocation();
            
            FVector Direction = AimTarget - BotCharacter->GetActorLocation();
            AimRotation = Direction.Rotation();
        }
    }
}

void UBotAnimInstance::UpdateLookAt()
{
    if (!BotCharacter) return;

    // Hedef varsa ona bak
    AAIController* AIController = Cast<AAIController>(BotCharacter->GetController());
    if (AIController)
    {
        AActor* FocusActor = AIController->GetFocusActor();
        if (FocusActor)
        {
            LookAtTarget = FocusActor->GetActorLocation() + FVector(0, 0, 50);
            LookAtWeight = 1.0f;
        }
        else
        {
            LookAtWeight = 0.0f;
        }
    }
}
```

---

## 9. Faz 7: Test ve Debug Araçları

### 9.1 Debug Visualizer Component

```cpp
// Source/BotAI/Public/Debug/AIDebugComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "AIDebugComponent.generated.h"

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class BOTAI_API UAIDebugComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UAIDebugComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType, 
        FActorComponentTickFunction* ThisTickFunction) override;

    // Debug toggles
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowBehaviorTree = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowUtilityAI = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowPerception = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowNavigation = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowDifficulty = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
    bool bShowIK = false;

protected:
    UPROPERTY(EditAnywhere, Category = "Debug")
    float TextScale = 1.0f;

    UPROPERTY(EditAnywhere, Category = "Debug")
    FColor DebugColor = FColor::Green;

private:
    void DrawBehaviorTreeInfo();
    void DrawUtilityAIInfo();
    void DrawPerceptionInfo();
    void DrawNavigationInfo();
    void DrawDifficultyInfo();
    void DrawIKInfo();

    class ABotAIController* GetAIController() const;
    class ABotCharacter* GetBotCharacter() const;
};
```

```cpp
// Source/BotAI/Private/Debug/AIDebugComponent.cpp
#include "Debug/AIDebugComponent.h"
#include "AI/BotAIController.h"
#include "Characters/BotCharacter.h"
#include "AI/UtilityAI/UtilityAIComponent.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "Perception/AIPerceptionComponent.h"
#include "DrawDebugHelpers.h"

UAIDebugComponent::UAIDebugComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UAIDebugComponent::TickComponent(float DeltaTime, ELevelTick TickType, 
    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (bShowBehaviorTree) DrawBehaviorTreeInfo();
    if (bShowUtilityAI) DrawUtilityAIInfo();
    if (bShowPerception) DrawPerceptionInfo();
    if (bShowNavigation) DrawNavigationInfo();
    if (bShowDifficulty) DrawDifficultyInfo();
    if (bShowIK) DrawIKInfo();
}

ABotAIController* UAIDebugComponent::GetAIController() const
{
    APawn* Pawn = Cast<APawn>(GetOwner());
    if (Pawn)
    {
        return Cast<ABotAIController>(Pawn->GetController());
    }
    return nullptr;
}

ABotCharacter* UAIDebugComponent::GetBotCharacter() const
{
    return Cast<ABotCharacter>(GetOwner());
}

void UAIDebugComponent::DrawBehaviorTreeInfo()
{
    ABotAIController* AIController = GetAIController();
    if (!AIController) return;

    UBlackboardComponent* BB = AIController->GetBlackboardComponent();
    if (!BB) return;

    FVector Location = GetOwner()->GetActorLocation() + FVector(0, 0, 200);

    // Blackboard değerlerini göster
    TArray<FString> DebugLines;
    DebugLines.Add(TEXT("=== BLACKBOARD ==="));
    
    // Target
    AActor* Target = Cast<AActor>(BB->GetValueAsObject(FName("TargetActor")));
    DebugLines.Add(FString::Printf(TEXT("Target: %s"), 
        Target ? *Target->GetName() : TEXT("None")));
    
    // Alert state
    bool bIsAlert = BB->GetValueAsBool(FName("IsAlert"));
    DebugLines.Add(FString::Printf(TEXT("Alert: %s"), 
        bIsAlert ? TEXT("YES") : TEXT("NO")));

    // Current action
    uint8 ActionValue = BB->GetValueAsEnum(FName("CurrentAction"));
    DebugLines.Add(FString::Printf(TEXT("Action: %d"), ActionValue));

    // Debug text çiz
    float YOffset = 0.0f;
    for (const FString& Line : DebugLines)
    {
        DrawDebugString(GetWorld(), Location + FVector(0, 0, -YOffset), 
            Line, nullptr, DebugColor, 0.0f, true, TextScale);
        YOffset += 20.0f;
    }
}

void UAIDebugComponent::DrawUtilityAIInfo()
{
    ABotCharacter* Bot = GetBotCharacter();
    if (!Bot) return;

    UUtilityAIComponent* UtilityAI = Bot->FindComponentByClass<UUtilityAIComponent>();
    if (!UtilityAI) return;

    FVector Location = GetOwner()->GetActorLocation() + FVector(100, 0, 150);

    // Utility AI skorlarını göster
    TMap<EBotAction, float> Scores = UtilityAI->GetActionScores();
    EBotAction CurrentAction = UtilityAI->GetCurrentAction();

    DrawDebugString(GetWorld(), Location, TEXT("=== UTILITY AI ==="), 
        nullptr, FColor::Yellow, 0.0f, true, TextScale);

    float YOffset = 20.0f;
    for (const auto& Pair : Scores)
    {
        FString ActionName = UEnum::GetValueAsString(Pair.Key);
        ActionName = ActionName.Right(ActionName.Len() - ActionName.Find(TEXT("::")) - 2);
        
        FString ScoreText = FString::Printf(TEXT("%s: %.2f"), *ActionName, Pair.Value);
        FColor Color = (Pair.Key == CurrentAction) ? FColor::Green : FColor::White;
        
        DrawDebugString(GetWorld(), Location + FVector(0, 0, -YOffset), 
            ScoreText, nullptr, Color, 0.0f, true, TextScale * 0.8f);
        YOffset += 15.0f;
    }
}

void UAIDebugComponent::DrawPerceptionInfo()
{
    ABotAIController* AIController = GetAIController();
    if (!AIController) return;

    UAIPerceptionComponent* Perception = AIController->GetAIPerceptionComponent();
    if (!Perception) return;

    FVector BotLocation = GetOwner()->GetActorLocation();

    // Sight range
    DrawDebugSphere(GetWorld(), BotLocation, 2000.0f, 32, 
        FColor::Blue, false, 0.0f, 0, 1.0f);

    // Hearing range
    DrawDebugSphere(GetWorld(), BotLocation, 1500.0f, 24, 
        FColor::Yellow, false, 0.0f, 0, 1.0f);

    // Algılanan aktörler
    TArray<AActor*> PerceivedActors;
    Perception->GetCurrentlyPerceivedActors(nullptr, PerceivedActors);

    for (AActor* Actor : PerceivedActors)
    {
        if (Actor)
        {
            DrawDebugLine(GetWorld(), BotLocation, Actor->GetActorLocation(), 
                FColor::Red, false, 0.0f, 0, 2.0f);
            DrawDebugSphere(GetWorld(), Actor->GetActorLocation(), 50.0f, 8, 
                FColor::Red, false, 0.0f, 0, 2.0f);
        }
    }
}

void UAIDebugComponent::DrawNavigationInfo()
{
    ABotAIController* AIController = GetAIController();
    if (!AIController) return;

    // Mevcut path'i çiz
    FNavPathSharedPtr Path = AIController->GetPathFollowingComponent()->GetPath();
    if (Path.IsValid())
    {
        const TArray<FNavPathPoint>& PathPoints = Path->GetPathPoints();
        
        for (int32 i = 0; i < PathPoints.Num() - 1; ++i)
        {
            DrawDebugLine(GetWorld(), 
                PathPoints[i].Location, 
                PathPoints[i + 1].Location, 
                FColor::Cyan, false, 0.0f, 0, 3.0f);
            
            DrawDebugSphere(GetWorld(), PathPoints[i].Location, 20.0f, 8, 
                FColor::Cyan, false, 0.0f, 0, 2.0f);
        }
    }
}

void UAIDebugComponent::DrawDifficultyInfo()
{
    ABotCharacter* Bot = GetBotCharacter();
    if (!Bot) return;

    FVector Location = GetOwner()->GetActorLocation() + FVector(-100, 0, 200);
    
    FBotStats Stats = Bot->GetBotStats();
    EBotDifficultyLevel Level = Bot->GetDifficultyLevel();

    TArray<FString> DebugLines;
    DebugLines.Add(TEXT("=== DIFFICULTY ==="));
    DebugLines.Add(FString::Printf(TEXT("Level: %s"), 
        *UEnum::GetValueAsString(Level)));
    DebugLines.Add(FString::Printf(TEXT("Health: %.0f/%.0f"), 
        Stats.CurrentHealth, Stats.MaxHealth));
    DebugLines.Add(FString::Printf(TEXT("Accuracy: %.0f%%"), 
        Stats.AimAccuracy * 100.0f));
    DebugLines.Add(FString::Printf(TEXT("Reaction: %.2fs"), 
        Stats.ReactionTime));
    DebugLines.Add(FString::Printf(TEXT("Aggression: %.0f%%"), 
        Stats.AggressionLevel * 100.0f));

    float YOffset = 0.0f;
    for (const FString& Line : DebugLines)
    {
        DrawDebugString(GetWorld(), Location + FVector(0, 0, -YOffset), 
            Line, nullptr, FColor::Orange, 0.0f, true, TextScale);
        YOffset += 15.0f;
    }
}

void UAIDebugComponent::DrawIKInfo()
{
    ABotCharacter* Bot = GetBotCharacter();
    if (!Bot) return;

    USkeletalMeshComponent* Mesh = Bot->GetMesh();
    if (!Mesh) return;

    // Ayak pozisyonlarını göster
    FVector LeftFoot = Mesh->GetSocketLocation(FName("foot_l"));
    FVector RightFoot = Mesh->GetSocketLocation(FName("foot_r"));

    DrawDebugSphere(GetWorld(), LeftFoot, 10.0f, 8, FColor::Green, false, 0.0f);
    DrawDebugSphere(GetWorld(), RightFoot, 10.0f, 8, FColor::Green, false, 0.0f);

    // IK trace'leri göster
    DrawDebugLine(GetWorld(), LeftFoot + FVector(0, 0, 50), 
        LeftFoot - FVector(0, 0, 100), FColor::Green, false, 0.0f, 0, 1.0f);
    DrawDebugLine(GetWorld(), RightFoot + FVector(0, 0, 50), 
        RightFoot - FVector(0, 0, 100), FColor::Green, false, 0.0f, 0, 1.0f);
}
```

### 9.2 Console Commands

```cpp
// Source/BotAI/Public/Debug/AIDebugCommands.h
#pragma once

#include "CoreMinimal.h"

class FAIDebugCommands
{
public:
    static void Register();
    static void Unregister();

private:
    static void ToggleAIDebug(const TArray<FString>& Args);
    static void SetDifficulty(const TArray<FString>& Args);
    static void SpawnBot(const TArray<FString>& Args);
    static void KillAllBots(const TArray<FString>& Args);
    static void PrintBotStats(const TArray<FString>& Args);
    static void ToggleRLTraining(const TArray<FString>& Args);
    
    static TArray<IConsoleCommand*> RegisteredCommands;
};
```

---

## 10. Demo Senaryoları

### 10.1 Senaryo 1: Utility AI Karar Verme Demo

**Amaç:** Botun farklı durumlarda dinamik karar verme sürecini göstermek.

**Kurulum:**
1. Tek bir bot ve bir oyuncu karakteri
2. Debug görselleştirme aktif
3. Utility AI skorları ekranda görünür

**Gösterilecekler:**
- Bot sağlıklıyken saldırgan davranış
- Hasar alınca siper arama
- Düşük sağlıkta kaçış
- Mermisi bitince reload

### 10.2 Senaryo 2: Adaptif Zorluk Demo

**Amaç:** Sistemin oyuncu performansına göre zorluğu nasıl ayarladığını göstermek.

**Kurulum:**
1. Difficulty meter UI
2. Oyuncu performans metrikleri görünür
3. Bot parametreleri anlık gösteriliyor

**Gösterilecekler:**
- Yüksek performanslı oyuncuda artan zorluk
- Sürekli ölen oyuncuda azalan zorluk
- Smooth transition (ani değişim yok)

### 10.3 Senaryo 3: RL Karşılaştırma Demo

**Amaç:** RL ile eğitilmiş bot ile rule-based bot arasındaki farkı göstermek.

**Kurulum:**
1. İki bot: biri RL, biri Behavior Tree
2. Aynı senaryoda performans karşılaştırması
3. TensorBoard eğitim grafikleri

**Gösterilecekler:**
- RL botunun adaptif davranışları
- Eğitim süreci ve reward grafikleri
- Her iki yaklaşımın güçlü/zayıf yönleri

---

## 11. Proje Yapısı ve Dosya Organizasyonu

```
Source/BotAI/
├── BotAI.Build.cs
├── BotAI.h
├── BotAI.cpp
│
├── Public/
│   ├── AI/
│   │   ├── BotAIController.h
│   │   ├── Tasks/
│   │   │   ├── BTTask_FindCover.h
│   │   │   ├── BTTask_Attack.h
│   │   │   └── BTTask_Patrol.h
│   │   ├── Decorators/
│   │   │   ├── BTDecorator_HasLineOfSight.h
│   │   │   └── BTDecorator_CheckHealth.h
│   │   ├── UtilityAI/
│   │   │   ├── UtilityAITypes.h
│   │   │   └── UtilityAIComponent.h
│   │   ├── DDA/
│   │   │   ├── PlayerPerformanceTracker.h
│   │   │   └── DifficultyManager.h
│   │   └── RL/
│   │       └── RLBridgeComponent.h
│   │
│   ├── Characters/
│   │   └── BotCharacter.h
│   │
│   ├── Animation/
│   │   └── BotAnimInstance.h
│   │
│   └── Debug/
│       ├── AIDebugComponent.h
│       └── AIDebugCommands.h
│
├── Private/
│   ├── AI/
│   │   ├── BotAIController.cpp
│   │   ├── Tasks/
│   │   │   ├── BTTask_FindCover.cpp
│   │   │   ├── BTTask_Attack.cpp
│   │   │   └── BTTask_Patrol.cpp
│   │   ├── Decorators/
│   │   │   ├── BTDecorator_HasLineOfSight.cpp
│   │   │   └── BTDecorator_CheckHealth.cpp
│   │   ├── UtilityAI/
│   │   │   └── UtilityAIComponent.cpp
│   │   ├── DDA/
│   │   │   ├── PlayerPerformanceTracker.cpp
│   │   │   └── DifficultyManager.cpp
│   │   └── RL/
│   │       └── RLBridgeComponent.cpp
│   │
│   ├── Characters/
│   │   └── BotCharacter.cpp
│   │
│   ├── Animation/
│   │   └── BotAnimInstance.cpp
│   │
│   └── Debug/
│       ├── AIDebugComponent.cpp
│       └── AIDebugCommands.cpp
│
Content/
├── AI/
│   ├── BT_BotMain.uasset (Behavior Tree)
│   ├── BB_BotData.uasset (Blackboard)
│   └── EQS/
│       ├── EQS_FindCover.uasset
│       └── EQS_FindFlankPosition.uasset
│
├── Characters/
│   └── Bot/
│       ├── BP_BotCharacter.uasset
│       ├── ABP_Bot.uasset (Anim Blueprint)
│       └── CR_Bot.uasset (Control Rig)
│
└── Debug/
    └── WBP_AIDebug.uasset (Debug Widget)

rl_server/
├── bot_ai_pb2.py
├── bot_ai_pb2_grpc.py
├── bot_rl_server.py
├── requirements.txt
└── models/
    └── bot_policy.zip
```

---

## Sonuç

Bu doküman, TÜBİTAK destekli projenin İş Paketi 2 kapsamındaki "Yapay Zeka Destekli Bot Yazılımı" MVP'sinin teknik implementasyon rehberini sunmaktadır.

**Önemli Notlar:**

1. Her faz birbirinin üzerine inşa edilmektedir. Sırayla ilerleyin.
2. Debug araçlarını erken aşamada aktif edin, problem tespitini kolaylaştırır.
3. TÜBİTAK hakem demosu için görselleştirme kritiktir.
4. Kod kalitesi ve dokümantasyon TÜBİTAK raporlaması için önemlidir.

---

*Doküman Sonu*
