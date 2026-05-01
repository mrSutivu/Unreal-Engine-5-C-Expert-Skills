---
name: multiplayer-seamless-travel-code
description: Complete C++ boilerplate for configuring Seamless Travel, GameMode transitions, and overriding APlayerState::CopyProperties.
---

# Multiplayer Seamless Travel Template (UE5 Reference)

## Activation / When to Use
- Trigger when implementing a Lobby-to-Match transition system without dropping network connections.
- Serves as the ultimate "Memory Dump" for `CopyProperties`.

## 1. Project Configuration
1. Create an empty map to act as the transition bridge (e.g., `Map_Transition`).
2. Go to **Project Settings -> Maps & Modes**.
3. Set the **Transition Map** to `Map_Transition`.

## 2. The GameMode Setup
Seamless Travel must be explicitly enabled in the GameMode of the originating map (the Lobby).

### `MyLobbyGameMode.h`
```cpp
#pragma once
#include "GameFramework/GameModeBase.h"
#include "MyLobbyGameMode.generated.h"

UCLASS()
class MYPROJECT_API AMyLobbyGameMode : public AGameModeBase
{
    GENERATED_BODY()
public:
    AMyLobbyGameMode();

    UFUNCTION(BlueprintCallable, Category = "Server")
    void StartMatchAndTravel();

    // Whitelist specific actors that must survive the travel (e.g., a persistent match manager)
    virtual void GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList) override;
};
```

### `MyLobbyGameMode.cpp`
```cpp
#include "MyLobbyGameMode.h"
#include "Engine/World.h"

AMyLobbyGameMode::AMyLobbyGameMode()
{
    // 1. ENABLE SEAMLESS TRAVEL
    bUseSeamlessTravel = true;
}

void AMyLobbyGameMode::StartMatchAndTravel()
{
    if (UWorld* World = GetWorld())
    {
        // 2. INITIATE SERVER TRAVEL
        // The boolean "true" signifies it is an absolute map path (not relative)
        World->ServerTravel(TEXT("/Game/Maps/Map_Arena?listen"), true);
    }
}

void AMyLobbyGameMode::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)
{
    Super::GetSeamlessTravelActorList(bToTransition, ActorList);

    // Add any global actors here that should not be destroyed.
    // NOTE: GameInstance, PlayerControllers, and PlayerStates are handled automatically.
}
```

## 3. The PlayerState Persistence (The Heart of Seamless Travel)
During travel, the old `PlayerState` is destroyed and a NEW `PlayerState` is spawned in the new map. You must manually copy the data.

### `MyPlayerState.h`
```cpp
#pragma once
#include "GameFramework/PlayerState.h"
#include "MyPlayerState.generated.h"

UCLASS()
class MYPROJECT_API AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    UPROPERTY(Replicated)
    int32 TeamID;

    UPROPERTY(Replicated)
    FName SelectedCharacterClass;

    // The function called during travel to pass data to the new instance
    virtual void CopyProperties(APlayerState* PlayerState) override;
    
    // Required to register replicated variables
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};
```

### `MyPlayerState.cpp`
```cpp
#include "MyPlayerState.h"
#include "Net/UnrealNetwork.h"

void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyPlayerState, TeamID);
    DOREPLIFETIME(AMyPlayerState, SelectedCharacterClass);
}

void AMyPlayerState::CopyProperties(APlayerState* PlayerState)
{
    // 1. MUST CALL SUPER FIRST (Copies core data like Ping, PlayerName, PlayerId)
    Super::CopyProperties(PlayerState);

    // 2. Cast to the NEW PlayerState instance
    if (AMyPlayerState* NewPS = Cast<AMyPlayerState>(PlayerState))
    {
        // 3. COPY DATA (Never copy raw Actor pointers!)
        NewPS->TeamID = this->TeamID;
        NewPS->SelectedCharacterClass = this->SelectedCharacterClass;
    }
}
```

## 4. The Controller Persistence
`APlayerController`s survive the travel by default. However, if you need to copy data when a completely *new* type of PlayerController is spawned for the new map, you must override `SeamlessTravelTo` or `CopyProperties` on the Controller. But standard practice is to keep volatile data in the `PlayerState` instead.
