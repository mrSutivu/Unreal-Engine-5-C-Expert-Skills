---
name: rpc-server-client-multicast
description: Authoritative implementation of Remote Procedure Calls (RPCs). Validates server authority, NetMulticast broadcasts, and Client ownership.
---

# Remote Procedure Calls (UE5 Expert)

## Activation / When to Use
- Mandatory when communicating across the network boundary between Server and Client.
- Trigger when sending inputs, applying damage, or broadcasting VFX/Sound.

## 1. The Three RPC Types
- **Server**: Called by a Client. Executes on the Server. Must be called from an Actor the Client OWNS (e.g., possessed Pawn or PlayerController).
- **Client**: Called by the Server. Executes ONLY on the specific Client that OWNS the Actor.
- **NetMulticast**: Called by the Server. Executes on the Server AND all connected Clients. (Good for effects, bad for state).

## 2. Implementation Syntax

### Header (.h)
`WithValidation` is recommended (mandatory in older UE, optional but good practice in UE5) for Server RPCs to prevent cheating.
```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    // SERVER: Client requests to sprint
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestSprint(bool bIsSprinting);

    // CLIENT: Server tells client their inventory is full
    UFUNCTION(Client, Unreliable)
    void Client_NotifyInventoryFull();

    // MULTICAST: Server tells everyone to play an explosion
    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_PlayExplosion(FVector Location);
};
```

### Source (.cpp)
You MUST implement `_Implementation`. For Server RPCs with validation, you MUST implement `_Validate`.
```cpp
// Validation runs BEFORE implementation. If it returns false, the client is kicked.
bool AMyCharacter::Server_RequestSprint_Validate(bool bIsSprinting)
{
    // E.g., check if the player is allowed to sprint (not stunned)
    return true; 
}

void AMyCharacter::Server_RequestSprint_Implementation(bool bIsSprinting)
{
    // Authorized logic runs here on the Server
    bSprinting = bIsSprinting;
}

void AMyCharacter::Client_NotifyInventoryFull_Implementation() { /* Update Local UI */ }
void AMyCharacter::Multicast_PlayExplosion_Implementation(FVector Location) { /* Spawn Particle */ }
```

## 3. Reliable vs Unreliable
- **Reliable**: Guaranteed delivery and ordered. High bandwidth cost. Use for critical state changes (e.g., `Server_EquipWeapon`).
- **Unreliable**: Fire and forget. Packet might drop. Use for frequent updates or non-critical VFX (e.g., `Multicast_PlayFootstep`).

## 4. Impact on Safety
- **Network Saturation**: Spamming Reliable Multicast RPCs on Tick will crash the server's network buffer. NEVER use RPCs on Tick.
- **Cheat Prevention**: Never trust parameters sent in a Server RPC. The Server must independently verify if the action is legal in `_Validate` or `_Implementation`.

## Common Mistakes (BAD vs GOOD)

**BAD (Trusting the Client)**:
```cpp
void AMyCharacter::Server_ApplyDamage_Implementation(AActor* Target, float DamageAmount) {
    // FATAL: A hacked client can send DamageAmount = 999999 to any target!
    Target->TakeDamage(DamageAmount, ...);
}
```

**GOOD (Server Authority)**:
```cpp
void AMyCharacter::Server_FireWeapon_Implementation() {
    // SAFE: Client just says "I fired". Server calculates hit and damage.
    if (Weapon->CanFire()) { Weapon->ServerFire(); }
}
```

## Verification Checklist
- [ ] `Server` RPCs are only called on Actors owned by the local connection.
- [ ] `NetMulticast` is NOT used for updating state (use `Replicated` variables instead).
- [ ] `Reliable` is used sparingly. `Unreliable` is used for visuals/audio.
- [ ] `_Implementation` suffix is used in the `.cpp` definitions.
- [ ] Server RPC inputs are validated against cheating.
