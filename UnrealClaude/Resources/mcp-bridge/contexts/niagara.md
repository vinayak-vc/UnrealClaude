# Unreal Engine 5 Niagara VFX Context

This context is automatically loaded when working with Niagara particle systems.

## Core Classes

### UNiagaraSystem
Top-level VFX asset. Contains one or more emitters. Referenced by path e.g. `/Game/VFX/NS_Burst`.

### UNiagaraComponent
Runtime component attached to an actor. Owns the simulation. Created by `SpawnSystemAtLocation`.

### UNiagaraFunctionLibrary
Static library for spawning Niagara systems at runtime.

```cpp
// Spawn at world location
UNiagaraComponent* Comp = UNiagaraFunctionLibrary::SpawnSystemAtLocation(
    World,
    NiagaraSystem,      // UNiagaraSystem*
    Location,           // FVector
    Rotation,           // FRotator
    Scale,              // FVector (default 1,1,1)
    bAutoDestroy,       // true = destroy when effect finishes
    bAutoActivate,      // true = start immediately
    ENCPoolMethod::None // No pooling
);

// Attach to actor/component
UNiagaraComponent* Comp = UNiagaraFunctionLibrary::SpawnSystemAttached(
    NiagaraSystem,
    AttachToComponent,
    AttachPointName,    // Socket/bone name
    Location,
    Rotation,
    EAttachLocation::KeepRelativeOffset,
    bAutoDestroy
);
```

## Setting Parameters at Runtime

### On UNiagaraComponent
```cpp
// Float parameter
NiagaraComp->SetVariableFloat(FName("MyFloat"), 42.0f);

// Vector parameter
NiagaraComp->SetVariableVec3(FName("MyVector"), FVector(1, 2, 3));

// LinearColor parameter
NiagaraComp->SetVariableLinearColor(FName("MyColor"), FLinearColor::Red);

// Bool parameter
NiagaraComp->SetVariableBool(FName("MyBool"), true);

// Int parameter
NiagaraComp->SetVariableInt(FName("MyInt"), 10);

// Object (texture, mesh, etc.)
NiagaraComp->SetVariableObject(FName("MyTexture"), SomeTexture);
```

### Common Parameter Naming Convention
Niagara parameters are typically prefixed: `User.MyParam` or just `MyParam`.
Check the Niagara System's User Exposed Parameters panel for exact names.

## Emitter Control
```cpp
// Activate/deactivate
NiagaraComp->Activate(/*bReset=*/true);
NiagaraComp->Deactivate();

// Force complete
NiagaraComp->DeactivateImmediate();

// Check state
bool bActive = NiagaraComp->IsActive();

// Set auto destroy
NiagaraComp->SetAutoDestroy(true);
```

## Pool Methods (ENCPoolMethod)
```cpp
ENCPoolMethod::None        // No pooling — always create new
ENCPoolMethod::AutoRelease // Return to pool when done (best performance)
ENCPoolMethod::ManualRelease // Return to pool manually
```
Use `AutoRelease` for frequently spawned effects (gunshots, footsteps).
Use `None` for rare one-shot effects (explosions).

## MCP Niagara Tool Operations

All operations via the `niagara` tool directly (not via `unreal_ue` router).

| Operation | Description | Key Params |
|-----------|-------------|------------|
| `spawn_system` | Spawn NiagaraSystem asset at world location | `system_path`, `location`, `rotation`, `scale`, `auto_destroy`, `actor_name` |
| `set_parameter` | Set a parameter on a spawned NiagaraComponent | `actor_name`, `param_name`, `param_type`, `param_value` |
| `list_systems` | Find NiagaraSystem assets in project | `package_path`, `name_filter`, `limit` |

### `param_type` values for `set_parameter`
| param_type | JSON value format |
|---|---|
| `float` | number |
| `int` | number |
| `bool` | boolean |
| `vector` | `{"x": 1, "y": 2, "z": 3}` |
| `color` | `{"r": 1.0, "g": 0.0, "b": 0.0, "a": 1.0}` (0-1 range) |

### Workflow: Find → Place → Tune

```json
// 1. Find available NS assets
{"tool": "niagara", "operation": "list_systems", "package_path": "/Game/VFX"}

// 2. Get geometric center of target mesh (for precise placement)
{"tool": "level", "operation": "get_actor_bounds", "actor_name": "SM_MyMesh"}
// → use 'origin' field from result as location

// 3. Spawn system at mesh center
{"tool": "niagara", "operation": "spawn_system",
 "system_path": "/Game/VFX/NS_Burst",
 "location": {"x": 0, "y": 0, "z": 100},
 "scale": {"x": 2, "y": 2, "z": 2},
 "auto_destroy": true}

// 4. Tune a parameter on the spawned actor
{"tool": "niagara", "operation": "set_parameter",
 "actor_name": "NS_Burst_0",
 "param_name": "SpawnRate",
 "param_type": "float",
 "param_value": 500.0}
```

## Stereoscopic / VR Depth Testing Tips
For stereoscopic depth pop testing (particles "coming out" of screen):
- Use **omnidirectional burst** with velocity pointing outward from center
- Set emitter to emit in sphere — covers all angles for stereo verification
- Spawn at mesh geometric center: `level get_actor_bounds` → use `origin`
- Scale 1.5-3x for visibility in both eyes at comfortable stereo depth
- `auto_destroy: true` for one-shot test bursts; `false` for looping review

## Best Practices
1. Use `auto_destroy: true` for one-shot effects (explosions, impacts)
2. Use pooling (`ENCPoolMethod::AutoRelease`) for frequent effects in C++
3. Name User-Exposed parameters clearly — Niagara parameter names are case-sensitive
4. `list_systems` before spawning to verify asset path exists
5. Check actor name from spawn result before calling `set_parameter`
