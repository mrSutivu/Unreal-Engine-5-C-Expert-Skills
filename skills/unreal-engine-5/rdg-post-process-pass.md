---
name: rdg-post-process-pass
description: Injecting a custom post-process pass using ISceneViewExtension and RDG. Modifies the final screen image via C++.
---

# Custom Post-Process Pass via RDG (UE5 Expert)

## Activation / When to Use
- Mandatory when creating complex screen-space effects that require C++ logic (e.g., custom Anti-Aliasing, procedural CRT filters, advanced outline systems).
- Trigger when Material Post-Process volumes are too limited.

## 1. The ISceneViewExtension Interface
To inject code into the Unreal rendering pipeline, you must implement `ISceneViewExtension`.

### Header (.h)
```cpp
#pragma once
#include "SceneViewExtension.h"

class FMyPostProcessExtension : public FSceneViewExtensionBase
{
public:
    FMyPostProcessExtension(const FAutoRegister& AutoRegister);

    // Required overrides
    virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {}
    virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {}
    virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override {}

    // The injection point (e.g., Post Tonemap)
    virtual void SubscribeToPostProcessingPass(EPostProcessingPass PassId, FAfterPassCallbackDelegateArray& InOutPassCallbacks, bool bIsPassEnabled) override;
};
```

## 2. Subscribing to the Pass
Inject your callback at the specific point in the pipeline (e.g., after the Tonemapper).

```cpp
void FMyPostProcessExtension::SubscribeToPostProcessingPass(EPostProcessingPass PassId, FAfterPassCallbackDelegateArray& InOutPassCallbacks, bool bIsPassEnabled)
{
    if (PassId == EPostProcessingPass::Tonemap)
    {
        InOutPassCallbacks.Add(FAfterPassCallbackDelegate::CreateRaw(this, &FMyPostProcessExtension::RenderMyPass));
    }
}
```

## 3. Building the RDG Pass
The callback provides the `FRDGBuilder` and the `FScreenPassTexture` representing the screen.

```cpp
FScreenPassTexture FMyPostProcessExtension::RenderMyPass(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& Inputs)
{
    // 1. Get the current screen texture
    FScreenPassTexture SceneColor = Inputs.Textures[(uint32)EPostProcessMaterialInput::SceneColor];

    // 2. Create an output texture for our pass
    FRDGTextureDesc OutputDesc = SceneColor.Texture->Desc;
    OutputDesc.Reset(); // Clear previous flags
    FRDGTextureRef OutputTexture = GraphBuilder.CreateTexture(OutputDesc, TEXT("MyPostProcessOutput"));

    // 3. Allocate parameters
    FMyPostProcessShaderParameters* PassParameters = GraphBuilder.AllocParameters<FMyPostProcessShaderParameters>();
    PassParameters->InputTexture = SceneColor.Texture;
    PassParameters->InputSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
    PassParameters->RenderTargets[0] = FRenderTargetBinding(OutputTexture, ERenderTargetLoadAction::ENoAction);

    // 4. Get the Global Shader
    TShaderMapRef<FMyPostProcessPixelShader> PixelShader(View.ShaderMap);

    // 5. Add the Draw pass
    FPixelShaderUtils::AddFullscreenPass(
        GraphBuilder,
        View.ShaderMap,
        RDG_EVENT_NAME("My Custom Post Process"),
        PixelShader,
        PassParameters,
        View.ViewRect
    );

    // 6. Return the new texture to continue down the pipeline
    return FScreenPassTexture(OutputTexture);
}
```

## 4. Impact on Safety
- **Pipeline Integrity**: Returning a properly constructed `FScreenPassTexture` ensures the engine can continue applying UI or other post-effects without crashing.
- **Modular Injection**: `ISceneViewExtension` allows plugins to add rendering features without modifying the engine source code.

## Verification Checklist
- [ ] Class inherits from `FSceneViewExtensionBase`.
- [ ] Injected pass uses `FPixelShaderUtils::AddFullscreenPass` for standard screen-space effects.
- [ ] The modified `FScreenPassTexture` is returned to the pipeline.
- [ ] `ISceneViewExtension` is registered and kept alive via a `TSharedPtr`.
