---
name: sequencer-cinematics-cpp
description: Controlling ALevelSequenceActor via C++. Play, Pause, and binding events to cinematic timelines.
---

# Sequencer & Cinematics (UE5 Expert)

## Activation / When to Use
- Mandatory when triggering cutscenes, complex environmental animations (e.g., a bridge collapsing), or camera fly-throughs.
- Trigger when you need C++ to know when a sequence finishes to resume gameplay.

## 1. Core Principles
The `ULevelSequence` is the asset. The `ALevelSequenceActor` is the entity in the world playing it. The `ULevelSequencePlayer` is the brain handling the playback state.

## 2. Implementation Syntax

### Playing a Sequence Dynamically
```cpp
#include "LevelSequence.h"
#include "LevelSequencePlayer.h"
#include "LevelSequenceActor.h"

void AMyGameManager::PlayCutscene(ULevelSequence* SequenceAsset)
{
    if (SequenceAsset)
    {
        FMovieSceneSequencePlaybackSettings Settings;
        Settings.bAutoPlay = false;
        Settings.LoopCount.Value = 0; // Play once
        Settings.bPauseAtEnd = true;

        ALevelSequenceActor* OutActor = nullptr;
        
        // Create the player and the actor simultaneously
        ULevelSequencePlayer* SequencePlayer = ULevelSequencePlayer::CreateLevelSequencePlayer(
            GetWorld(), 
            SequenceAsset, 
            Settings, 
            OutActor
        );

        if (SequencePlayer)
        {
            // Bind completion delegate
            SequencePlayer->OnFinished.AddDynamic(this, &AMyGameManager::OnCutsceneFinished);
            
            // Start playback
            SequencePlayer->Play();
        }
    }
}
```

### Resuming Gameplay
```cpp
void AMyGameManager::OnCutsceneFinished()
{
    // Return control to player, hide cinematic black bars, etc.
    UE_LOG(LogTemp, Log, TEXT("Cutscene ended. Resuming game."));
}
```

## 3. Passing Actors to Sequencer (Binding)
Sequencer can animate actors in the world. C++ can dynamically tell Sequencer *which* actor to animate (e.g., grabbing the actual Player Pawn instead of a dummy).

```cpp
void BindPlayerToSequence(ULevelSequencePlayer* Player, ALevelSequenceActor* SeqActor, AActor* ActorToBind, FName TrackName)
{
    // Find the track by name
    TArray<FMovieSceneBinding> Bindings = SeqActor->GetSequence()->GetMovieScene()->GetBindings();
    for (const FMovieSceneBinding& Binding : Bindings)
    {
        if (Binding.GetName() == TrackName.ToString())
        {
            // Override the binding to use our specific Actor
            FMovieSceneObjectBindingID BindingID = FMovieSceneObjectBindingID(Binding.GetObjectGuid());
            Player->BindJumpToEvent(BindingID); // Legacy/Alternative
            
            // Modern Override
            SeqActor->AddBindingByTag(TrackName, ActorToBind);
            break;
        }
    }
}
```

## 4. Impact on Safety
- **Garbage Collection**: `CreateLevelSequencePlayer` allocates an Actor. If `Settings.bRestoreState` is false, and the actor isn't destroyed manually when finished, it leaves a dormant actor in the world.
- **Network Sync**: Sequencer playback does NOT natively replicate perfectly frame-by-frame. For multiplayer, rely on Multicast RPCs to tell all clients to `Play()` simultaneously, or use specific Networked Sequencer plugins.

## Verification Checklist
- [ ] `LevelSequence` and `MovieScene` modules are added to `Build.cs`.
- [ ] `ULevelSequencePlayer::CreateLevelSequencePlayer` is used for dynamic instancing.
- [ ] `OnFinished` delegate is bound to resume gameplay securely.
- [ ] `bPauseAtEnd` or `bRestoreState` are configured intentionally.
