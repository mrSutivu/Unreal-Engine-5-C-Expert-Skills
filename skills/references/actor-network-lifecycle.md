---
name: actor-network-lifecycle
description: Chronological reference mapping the precise initialization and replication order of an AActor and its components.
---

# Actor Network Lifecycle (UE5 Expert Reference)

## Activation / When to Use
- Trigger when debugging "Race Conditions" where a client tries to read a pointer that hasn't arrived over the network yet.
- Trigger when deciding whether to put logic in `PostInitializeComponents` or `BeginPlay`.

## 1. The Server Timeline (Authority)
The server dictates creation. Its lifecycle is deterministic and synchronous.

1. **`SpawnActor` called** (Memory allocated).
2. **`PreInitializeComponents`**
3. **`InitializeComponent`** (Called sequentially on every `UActorComponent` attached).
4. **`PostInitializeComponents`** (Components are fully setup, but Game has not officially started).
5. **`BeginPlay`** (The Actor officially joins the game logic).
6. **`Tick`** (Starts processing frames).

## 2. The Client Timeline (Replication)
The client timeline is **ASYNCHRONOUS**. Replication packets do not guarantee order.

1. **Server replicates Spawn** -> Client creates an empty hull (`AActor`).
2. **`PostInitializeComponents`** (Components exist locally, but REPLICATED VALUES ARE NOT YET HERE).
3. **Server replicates `PlayerState` / `PlayerController`** (Might arrive now, might arrive later).
4. **`BeginPlay`** (Executes on client. **WARNING**: Replicated variables might STILL be default values!).
5. **Server replicates properties** -> Replicated variables are filled.
6. **`OnRep_...` functions fire** -> (This is the FIRST time you know the true Server value).
7. **`OnRep_PlayerState` / `OnRep_Controller` fire** (For Pawns).

## 3. The "Player Join" Network Order
When a Client connects to a dedicated server, here is the order in which major framework classes arrive on the local machine:

1. **`UGameInstance`** (Already exists locally).
2. **`APlayerController`** (The client's own controller is created locally, then syncs).
3. **`AGameStateBase`** (Replicated to client. Contains global data).
4. **`APlayerState`** (Replicated to client. Populated into GameState's PlayerArray).
5. **`APawn`** (Replicated to client. Spawns into world).
6. **`Possess`** -> Client's PlayerController takes control of the Pawn.

## 4. The Golden Rules for Initialization

### RULE 1: Never trust `BeginPlay` for replicated pointers on a Client.
If you need to access your `PlayerState` to set up UI, do NOT do it in `BeginPlay`.
```cpp
// BAD
void AMyPawn::BeginPlay() {
    UpdateUI(GetPlayerState()->Score); // CRASH on client: PlayerState hasn't replicated yet!
}

// GOOD
void AMyPawn::OnRep_PlayerState() {
    UpdateUI(GetPlayerState()->Score); // SAFE: PlayerState is guaranteed to exist now.
}
```

### RULE 2: `PostInitializeComponents` vs `BeginPlay`
- Use `PostInitializeComponents` to bind internal delegates between components (e.g., binding a `UHealthComponent`'s `OnDeath` delegate to the Actor). This ensures the bindings exist before the Server replicates any damage.
- Use `BeginPlay` to query the world or interact with other Actors.

### RULE 3: RPC Safety
You CANNOT call a Server RPC from a Client until the Client's `APlayerController` has finished possessing the `APawn`. If an RPC fires too early (e.g., during `BeginPlay` of the Pawn), the Engine drops it with a "No owning connection" warning.

## 5. The RepNotify `PostInitProperties` Trick
If you need a RepNotify to fire instantly when an Actor spawns on a client (to set initial visual state), you don't need to manually call it.
- **Fact**: `OnRep` functions ONLY fire if the replicated value is DIFFERENT from the client's current local value.
- **Trick**: If the Server spawns an Actor with `TeamColor = Red`, and the Client's C++ default is `TeamColor = Blue`, the `OnRep_TeamColor` WILL fire. If both default to `Red`, the `OnRep` WILL NOT fire. To guarantee execution, initialize the local default to an invalid state (e.g., `TeamColor = None`).
