---
name: farchive-binary-serialization
description: Low-level binary serialization using FArchive. Writing bytes directly to memory or disk for custom networking or save formats.
---

# FArchive Binary Serialization (UE5 Expert)

## Activation / When to Use
- Mandatory when bypassing `USaveGame` for custom file formats, network packets, or extreme performance.
- Trigger when you need to serialize complex byte arrays or custom C++ classes that don't inherit from `UObject`.

## 1. Core Principles
`FArchive` is a stream. It uses the overloaded `<<` operator to represent BOTH Reading and Writing, depending on the archive's mode.

## 2. Implementation Syntax

### Serializing a Struct
Define how your struct is packed into bytes.

```cpp
struct FMyCustomData
{
    int32 ID;
    FString Name;
    FVector Location;

    // The magical serialize function
    friend FArchive& operator<<(FArchive& Ar, FMyCustomData& Data)
    {
        Ar << Data.ID;
        Ar << Data.Name;
        Ar << Data.Location;
        return Ar;
    }
};
```

### Writing to Disk
Use `FMemoryWriter` and `FFileHelper`.

```cpp
void SaveToBinaryFile(const FString& FilePath, FMyCustomData& DataToSave)
{
    TArray<uint8> ByteBuffer;
    FMemoryWriter MemoryWriter(ByteBuffer, true);

    // This calls the operator<< we defined, packing the struct into the ByteBuffer
    MemoryWriter << DataToSave;

    // Write bytes to disk
    FFileHelper::SaveArrayToFile(ByteBuffer, *FilePath);
}
```

### Reading from Disk
Use `FMemoryReader`.

```cpp
void LoadFromBinaryFile(const FString& FilePath, FMyCustomData& OutLoadedData)
{
    TArray<uint8> ByteBuffer;
    if (FFileHelper::LoadFileToArray(ByteBuffer, *FilePath))
    {
        FMemoryReader MemoryReader(ByteBuffer, true);

        // This calls the exact same operator<<, but because it's a Reader, it unpacks the bytes into the struct!
        MemoryReader << OutLoadedData;
    }
}
```

## 3. Impact on Safety
- **Order Dependency**: The `<<` operations MUST be perfectly identical during Read and Write. If you write `ID`, `Name`, `Location`, but read `Name`, `ID`, `Location`, the memory will be corrupted and crash.
- **Versioning**: Unlike `UObject` serialization which handles missing variables gracefully, raw `FArchive` breaks if the struct changes. You MUST manually write a version integer at the start of the archive to handle backward compatibility.

## Common Mistakes (BAD vs GOOD)

**BAD (Changing struct without versioning)**:
```cpp
// V1: Ar << ID;
// V2: Ar << ID; Ar << Health; // CRASHES when reading a V1 file!
```

**GOOD (Manual Versioning)**:
```cpp
int32 Version = 2;
Ar << Version;
Ar << Data.ID;
if (Version >= 2) { Ar << Data.Health; } // Safe backward compatibility
```

## Verification Checklist
- [ ] `operator<<` is overloaded for custom structs.
- [ ] Variables are serialized in the EXACT same order.
- [ ] Versioning integers are injected at the start of the archive.
