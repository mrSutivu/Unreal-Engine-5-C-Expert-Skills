---
name: async-multithreading-tasks
description: Complete C++ templates for UE5 asynchronous operations. Covers AsyncTask, FRunnable, TaskGraph, and UE5Coro.
---

# Async & Multi-Threading Templates (UE5 Expert Reference)

## Activation / When to Use
- Trigger when you need the EXACT boilerplate to push heavy mathematical, procedural, or I/O calculations off the Game Thread.
- Serves as the ultimate "Memory Dump" for CPU parallelism in Unreal Engine 5.

## 1. AsyncTask (The Simplest Approach)
Use for "Fire and Forget" tasks or when you just need to run a small block of code in the background and return to the Game Thread.

```cpp
#include "Async/Async.h"

void AMyActor::CalculateProceduralMesh()
{
    // 1. Capture variables safely (By value for primitives, TWeakObjectPtr for UObjects)
    TWeakObjectPtr<AMyActor> WeakThis(this);
    int32 HeavyIterations = 100000;

    // 2. Push to Background Thread
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [WeakThis, HeavyIterations]()
    {
        // ---> We are now on a Worker Thread <---
        // DO NOT CREATE UOBJECTS OR SPAWN ACTORS HERE.
        
        float Result = 0.f;
        for (int32 i = 0; i < HeavyIterations; ++i)
        {
            Result += FMath::Sin(i); // Heavy math
        }

        // 3. Return to Game Thread to apply results
        AsyncTask(ENamedThreads::GameThread, [WeakThis, Result]()
        {
            // ---> We are back on the Game Thread <---
            // Safe to modify Actors and UObjects
            if (WeakThis.IsValid())
            {
                WeakThis->ApplyProceduralResult(Result);
            }
        });
    });
}
```

## 2. FRunnable (The Dedicated Thread)
Use when you need a persistent background thread that runs continuously (e.g., a custom TCP server listener, or a continuous physics sim).

### Header (`MyWorker.h`)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "HAL/Runnable.h"

class FMyWorker : public FRunnable
{
public:
    FMyWorker();
    virtual ~FMyWorker();

    // FRunnable interface
    virtual bool Init() override;
    virtual uint32 Run() override;
    virtual void Stop() override;
    virtual void Exit() override;

    void EnsureCompletion();

private:
    FRunnableThread* Thread;
    FThreadSafeBool bStopThread;
};
```

### Source (`MyWorker.cpp`)
```cpp
#include "MyWorker.h"
#include "HAL/RunnableThread.h"

FMyWorker::FMyWorker() : bStopThread(false)
{
    // Create the thread immediately
    Thread = FRunnableThread::Create(this, TEXT("MyCustomWorkerThread"), 0, TPri_BelowNormal);
}

FMyWorker::~FMyWorker()
{
    if (Thread) {
        EnsureCompletion();
        delete Thread;
        Thread = nullptr;
    }
}

bool FMyWorker::Init() { return true; }

uint32 FMyWorker::Run()
{
    // The continuous loop
    while (!bStopThread)
    {
        // Do heavy persistent work here...
        
        // Yield to prevent 100% CPU lock
        FPlatformProcess::Sleep(0.01f);
    }
    return 0;
}

void FMyWorker::Stop()
{
    bStopThread = true;
}

void FMyWorker::Exit() { }

void FMyWorker::EnsureCompletion()
{
    Stop();
    if (Thread) {
        Thread->WaitForCompletion(); // Blocks until thread safely exits
    }
}
```

## 3. UE::Tasks (The Modern UE5 Task Graph)
Replaces the old `FGraphEvent`. Used for launching tasks that return values or depend on other tasks (Prerequisites).

```cpp
#include "Tasks/Task.h"

void AMyManager::ProcessDataChunk()
{
    // 1. Launch a task that returns a complex struct
    UE::Tasks::FTask<FMyComplexData> DataTask = UE::Tasks::Launch(
        TEXT("ProcessDataChunk"), 
        []() -> FMyComplexData 
        {
            FMyComplexData ProcessedData;
            // ... heavy processing ...
            return ProcessedData;
        }
    );

    // 2. Launch a continuation task that ONLY runs when DataTask finishes
    UE::Tasks::Launch(
        TEXT("ApplyDataChunk"), 
        [DataTask]() 
        {
            // Get the result from the previous task safely
            FMyComplexData Result = DataTask.GetResult();
            
            // ... process result ...
        }, 
        DataTask // This is the Prerequisite
    );
}
```

## 4. UE5Coro (C++20 Coroutines - Advanced)
If using the `UE5Coro` plugin or native UE5.3+ coroutine support. Allows writing async code that looks synchronous.

```cpp
#include "UE5Coro.h"

UE5Coro::TCoroutine<> AMyActor::PerformAsyncSequence()
{
    UE_LOG(LogTemp, Log, TEXT("Step 1: Game Thread"));

    // Move to background thread instantly
    co_await UE5Coro::Async::MoveToBackground();

    // ---> Now on Worker Thread <---
    float CalculatedData = PerformHeavyMath();

    // Move back to Game Thread
    co_await UE5Coro::Async::MoveToGameThread();

    // ---> Now on Game Thread <---
    ApplyData(CalculatedData);
}
```
