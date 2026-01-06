# Unreal Engine 5 - Sky Atmosphere Implementation Notes

Research conducted 2026-01-06 using UE 5.6 and 5.7.1 source.

## Is it a mesh?

**Desktop/PC: No mesh** - Uses a procedurally generated **fullscreen triangle** (`DrawPrimitive(0, 1, 1)`) for ray marching.

**Mobile: Yes, requires a skydome mesh** - Due to performance constraints, mobile requires a user-provided skydome mesh with a sky material that samples the atmosphere LUTs.

## What mesh (when used)?

On mobile or for custom skydome setups, you use any mesh that covers the sky (typically a sphere or inverted dome). The mesh shape matters because it drives the world position for atmosphere sampling. There's an example in:
```
Engine/Content/Maps/Templates/TimeOfDay_Default
```

## Does it use a material?

**Core rendering: No traditional material** - The atmosphere is rendered via dedicated shaders, not a material graph.

**Skydome workflow: Yes** - When using a skydome mesh, you create a material using these expressions:
- `SkyAtmosphereViewLuminance` - Sky color
- `SkyAtmosphereAerialPerspective` - Fog/haze (RGB=luminance, A=transmittance)
- `SkyAtmosphereLightDiskLuminance` - Sun disk
- `SkyAtmosphereLightIlluminance` - Light on clouds
- `SkyAtmosphereLightDirection` - Light vector

## Shaders Used

| Type | Shader | Purpose |
|------|--------|---------|
| Vertex | `SkyAtmosphereVS` | Fullscreen quad setup |
| Pixel | `RenderSkyAtmosphereRayMarchingPS` | Main sky ray marching |
| Compute | `RenderTransmittanceLutCS` | 256x64 transmittance LUT |
| Compute | `RenderMultiScatteredLuminanceLutCS` | 32x32 multi-scatter LUT |
| Compute | `RenderSkyViewLutCS` | 192x104 sky view LUT |
| Compute | `RenderCameraAerialPerspectiveVolumeCS` | 32x32x16 aerial perspective 3D texture |

Shader file: `/Engine/Private/SkyAtmosphere.usf`

## LUT Textures (5 total)

| LUT | Size | Purpose |
|-----|------|---------|
| Transmittance | 256x64 | Optical depth lookup (height/angle -> exp(-extinction)) |
| MultiScatteredLuminance | 32x32 | Higher-order scattering approximation |
| SkyView (FastSky) | 192x104 | Pre-computed sky luminance (lat/long) |
| CameraAerialPerspectiveVolume | 32x32x16 | 3D froxel volume for aerial perspective |
| DistantSkyLight | Buffer (1 elem) | Ambient sky illuminance for clouds |

## Rendering Pipeline

1. **LUT Generation** (compute shaders): Pre-compute transmittance, multi-scattering, sky view, and aerial perspective into textures
2. **Fast Path**: Sample LUTs directly for most pixels (cheap, but can cause artifacts - see Space View Issues)
3. **Ray March Fallback**: Full physical simulation when needed - variable sample count, Rayleigh+Mie+Ozone scattering, planet shadows
4. **Compositing**: Blends onto scene color with pre-multiplied alpha before BasePass

## Ray Marching Details

- **Ray Setup**: Camera pos to atmosphere top/bottom intersection, capped at 9000km
- **Sample Count**: Variable (lerp by distance), quadratic spacing + per-pixel noise
- **Per-Sample**:
  1. Sample medium (Rayleigh/Mie/Ozone extinction & scattering)
  2. Compute transmittance to light via LUT
  3. Apply planet shadow, VSM, cloud shadows
  4. Phase functions: Rayleigh (3/4(1+cos²θ)), Mie (Henyey-Greenstein)
  5. Accumulate in-scattered luminance + multi-scattering

### Depth Buffer Integration

For opaque geometry, the ray marcher reads the depth buffer to limit ray distance. This is where edge artifacts originate.

**Function:** `IntegrateSingleScatteredLuminance()` (SkyAtmosphere.usf:399)

```hlsl
// SkyAtmosphere.usf lines 491-512
// Inside IntegrateSingleScatteredLuminance(), within #if VIEWDATA_AVAILABLE block

float PlanetOnOpaque = 1.0f;  // This is used to hide opaque meshes under the planet ground
#if VIEWDATA_AVAILABLE
#if SAMPLE_ATMOSPHERE_ON_CLOUDS
    // ... cloud path uses DeviceZ directly as world distance ...
#else // SAMPLE_ATMOSPHERE_ON_CLOUDS
    if (DeviceZ != FarDepthValue)
    {
        const float3 DepthBufferTranslatedWorldPosKm = GetScreenTranslatedWorldPos(SVPos, DeviceZ).xyz * CM_TO_SKY_UNIT;
        const float3 TraceStartTranslatedWorldPosKm = WorldPos + View.SkyPlanetTranslatedWorldCenterAndViewHeight.xyz * CM_TO_SKY_UNIT;
        const float3 TraceStartToSurfaceWorldKm = DepthBufferTranslatedWorldPosKm - TraceStartTranslatedWorldPosKm;
        float tDepth = length(TraceStartToSurfaceWorldKm);  // Line 496 - Issue: length() includes perpendicular error
        if (tDepth < tMax)
        {
            tMax = tDepth;
        }
        else
        {
            // Artists did not like that we handle automatic hiding of opaque element behind the planet.
            // Now, pixel under the surface of earht will receive aerial perspective as if they were on the ground.
            //PlanetOnOpaque = 0.0;
        }

        // if the ray intersects with the atmosphere boundary, make sure we do not apply atmosphere on surfaces are front of it.
        if (dot(WorldDir, TraceStartToSurfaceWorldKm) < 0.0)  // Line 509 - Issue: No epsilon tolerance
        {
            return Result;  // Early exit with zero luminance - causes dark edges
        }
    }
#endif // SAMPLE_ATMOSPHERE_ON_CLOUDS
#endif
```

### Helper Function: GetScreenTranslatedWorldPos

**Function:** `GetScreenTranslatedWorldPos()` (SkyAtmosphere.usf:161)

```hlsl
// SkyAtmosphere.usf lines 161-167
float4 GetScreenTranslatedWorldPos(float4 SVPos, float DeviceZ)
{
#if HAS_INVERTED_Z_BUFFER
    DeviceZ = max(0.000000000001, DeviceZ);  // TODO: investigate why SvPositionToWorld returns bad values when DeviceZ is far=0 when using inverted z
#endif
    return float4(SvPositionToTranslatedWorld(float4(SVPos.xy, DeviceZ, 1.0)), 1.0);
}
```

### Ray-Sphere Intersection

**Function:** `RayIntersectSphere()` (Common.ush:1952)

```hlsl
// Common.ush lines 1948-1975
/**
 * Returns near intersection in x, far intersection in y, or both -1 if no intersection.
 * RayDirection does not need to be unit length.
 */
float2 RayIntersectSphere(float3 RayOrigin, float3 RayDirection, float4 Sphere)
{
    float3 LocalPosition = RayOrigin - Sphere.xyz;
    float LocalPositionSqr = dot(LocalPosition, LocalPosition);

    float3 QuadraticCoef;
    QuadraticCoef.x = dot(RayDirection, RayDirection);
    QuadraticCoef.y = 2 * dot(RayDirection, LocalPosition);
    QuadraticCoef.z = LocalPositionSqr - Sphere.w * Sphere.w;

    float Discriminant = QuadraticCoef.y * QuadraticCoef.y - 4 * QuadraticCoef.x * QuadraticCoef.z;

    float2 Intersections = -1;

    // Only continue if the ray intersects the sphere
    FLATTEN
    if (Discriminant >= 0)  // Line 1968 - Issue: No epsilon for near-zero cases (grazing rays)
    {
        float SqrtDiscriminant = sqrt(Discriminant);
        Intersections = (-QuadraticCoef.y + float2(-1, 1) * SqrtDiscriminant) / (2 * QuadraticCoef.x);
    }

    return Intersections;
}
```

### Precision Constants

**File:** SkyAtmosphereCommon.ush (lines 24-30)

```hlsl
// Float accuracy offset in Sky unit (km, so this is 1m). Should match the one in FAtmosphereSetup::ComputeViewData
#define PLANET_RADIUS_OFFSET 0.001f

// Planet radius safe edge to make sure ray does intersect with the atmosphere, for it to traverse the atmosphere.
// Must match the one in FSceneRenderer::RenderSkyAtmosphereInternal.
// This is (0.01km/6420km).
#define PLANET_RADIUS_RATIO_SAFE_EDGE 1.00000155763f
```

**Where these constants are actually used:**

| Constant | Location | Usage |
|----------|----------|-------|
| `PLANET_RADIUS_OFFSET` | SkyAtmosphere.usf:192 | `MoveToTopAtmosphere()` - offset when entering atmosphere |
| `PLANET_RADIUS_OFFSET` | SkyAtmosphere.usf:664 | `IntegrateSingleScatteredLuminance()` - planet shadow for light 0 |
| `PLANET_RADIUS_OFFSET` | SkyAtmosphere.usf:702 | `IntegrateSingleScatteredLuminance()` - planet shadow for light 1 |
| `PLANET_RADIUS_RATIO_SAFE_EDGE` | SkyAtmosphere.usf:955 | `RenderSkyAtmosphereRayMarchingPS()` - FastSky height check |

**Note:** Neither constant is used in the depth buffer integration code where the edge artifacts occur.

## Physical Model

| Component | Scattering | Absorption | Density Profile |
|-----------|------------|------------|-----------------|
| Rayleigh | `RayleighScattering * exp(scale*h)` | 0 | Exponential falloff |
| Mie | `MieScattering * exp(scale*h)` | `MieAbsorption * density` | Steeper exponential |
| Ozone | 0 | `AbsorptionExtinction * linear` | Bi-linear (0-20km up, 20-50km down) |

- Planet radius: ~6360km (Earth-like default)
- Atmosphere height: ~60km
- Units: km internally (`CM_TO_SKY_UNIT = 1e-5`)

## Space View Issues & Fixes

The system is optimized for ground-level views. When viewing a planet from space, several artifacts can occur.

### Edge/Limb Darkening Artifacts

**Symptom:** Dark shadow-like artifacts at planet edges where mesh vertices appear to "poke through" the atmosphere. Center looks fine, edges have problems.

**Confirmed in:** UE 5.6, 5.7.1

### All Contributing Factors (Deep Dive Analysis)

| Factor | Function | File:Line | Code Context |
|--------|----------|-----------|--------------|
| **Dot check - no epsilon** | `IntegrateSingleScatteredLuminance()` | SkyAtmosphere.usf:509 | `if (dot(WorldDir, TraceStartToSurfaceWorldKm) < 0.0)` |
| **Length vs projected distance** | `IntegrateSingleScatteredLuminance()` | SkyAtmosphere.usf:496 | `float tDepth = length(TraceStartToSurfaceWorldKm);` |
| **Ray-sphere discriminant** | `RayIntersectSphere()` | Common.ush:1968 | `if (Discriminant >= 0)` |
| **Radius offset unused** | (definition only) | SkyAtmosphereCommon.ush:24 | `#define PLANET_RADIUS_OFFSET 0.001f` |
| **Planar depth test** | C++ renderer setup | SkyAtmosphereRendering.cpp | `StartDepthZ` from center ray |
| **Inverted Z precision** | `GetScreenTranslatedWorldPos()` | SkyAtmosphere.usf:164 | `DeviceZ = max(0.000000000001, DeviceZ);` |

### Epic's Own TODOs Acknowledging FP Issues

**TODO 1:** `GetScreenTranslatedWorldPos()` (SkyAtmosphere.usf:164)
```hlsl
DeviceZ = max(0.000000000001, DeviceZ);  // TODO: investigate why SvPositionToWorld returns bad values when DeviceZ is far=0 when using inverted z
```

**TODO 2:** `RenderCameraAerialPerspectiveVolumeCS()` (SkyAtmosphere.usf:1529)
```hlsl
if (BelowHorizon || UnderGround)
{
    CamPos += normalize(CamPos) * 0.02f;  // TODO: investigate why we need this workaround. Without it, we get some bad color and flickering on the ground only (floating point issue with sphere intersection code?).

    float3 VoxelWorldPosNorm = normalize(VoxelWorldPos);
    float3 CamProjOnGround = normalize(CamPos) * Atmosphere.BottomRadiusKm;
    // ... workaround continues with horizon detection ...
}
```

### Alternative Code Paths (Bypass Options)

| Path | Define | Affected by Bug? |
|------|--------|------------------|
| **FastSky LUT** | `FASTSKY_ENABLED` | No - bypasses ray marching entirely |
| **Fast Aerial Perspective** | `FASTAERIALPERSPECTIVE_ENABLED` | No - uses 3D LUT |
| **Clouds Path** | `SAMPLE_ATMOSPHERE_ON_CLOUDS` | Yes - still calls `IntegrateSingleScatteredLuminance` |
| **Depth Disabled** | `DepthReadDisabled` | No - forces `DeviceZ = FarDepthValue` |

### Console Variable Fixes

**Recommended combo (used by Cesium for Unreal):**
```
r.SkyAtmosphere.FastSkyLUT 0
r.SkyAtmosphere.AerialPerspectiveLUT.FastApplyOnOpaque 0
```

**Full troubleshooting set:**

| Command | Effect | Performance |
|---------|--------|-------------|
| `r.SkyAtmosphere.FastSkyLUT 0` | Disables LUT optimization, per-pixel ray march | Slower |
| `r.SkyAtmosphere.AerialPerspectiveLUT.FastApplyOnOpaque 0` | Disables fast aerial perspective | Slower |
| `r.SkyAtmosphere.AerialPerspective.DepthTest 0` | Disables planar depth test | Slower |
| `r.SkyAtmosphere.SampleCountMax 64` | More ray samples (default ~32) | Slower |
| `r.SkyAtmosphere.DistanceToSampleCountMax 1000` | Extend sample distance | Minor |
| `r.SkyAtmosphere.AerialPerspectiveLUT.DepthResolution 32` | Higher depth slice resolution | Minor |
| `r.SkyAtmosphere.FastSkyLUT.SampleCountMax 64` | Higher LUT sample count | Minor |

### Directional Light Settings for Space

On your sun directional light, enable:
- **Per Pixel Atmosphere Transmittance** = True (critical for correct planetary shadowing)
- **Cast Shadows on Atmosphere** = True
- **Atmosphere Sun Light Index** = 0

### Component Settings

| Property | Recommendation | Why |
|----------|----------------|-----|
| `Aerial Perspective Start Depth` | Set > planet radius for space views | Pushes AP start beyond mesh surface |
| `Bottom Radius` | Match exactly to mesh radius | Eliminates radius mismatch causing clipping |
| `Transform Mode` | `Planet Center at Component Transform` | Explicit positioning |

### Shader Fixes (Engine Modification)

#### Option 1: Minimal - Add Epsilon to Dot Check

```hlsl
// SkyAtmosphere.usf line 509
// In IntegrateSingleScatteredLuminance()

// Replace:
if (dot(WorldDir, TraceStartToSurfaceWorldKm) < 0.0)

// With:
if (dot(WorldDir, TraceStartToSurfaceWorldKm) < -0.001)  // 1 meter tolerance in km
```

#### Option 2: Recommended - Use Projected Distance

Fixes both length precision and dot check in one change:

```hlsl
// SkyAtmosphere.usf lines 491-515
// In IntegrateSingleScatteredLuminance(), replace the #else // SAMPLE_ATMOSPHERE_ON_CLOUDS block:

#else // SAMPLE_ATMOSPHERE_ON_CLOUDS
    if (DeviceZ != FarDepthValue)
    {
        const float3 DepthBufferTranslatedWorldPosKm = GetScreenTranslatedWorldPos(SVPos, DeviceZ).xyz * CM_TO_SKY_UNIT;
        const float3 TraceStartTranslatedWorldPosKm = WorldPos + View.SkyPlanetTranslatedWorldCenterAndViewHeight.xyz * CM_TO_SKY_UNIT;
        const float3 TraceStartToSurfaceWorldKm = DepthBufferTranslatedWorldPosKm - TraceStartTranslatedWorldPosKm;

        // Use projected distance (robust to perpendicular FP noise)
        float tDepth = dot(WorldDir, TraceStartToSurfaceWorldKm);

        // Early exit if surface is behind ray start (with epsilon for FP tolerance)
        if (tDepth < 1e-4)
        {
            return Result;
        }

        if (tDepth < tMax)
        {
            tMax = tDepth;
        }
        // Removed redundant dot check - already handled above
    }
#endif // SAMPLE_ATMOSPHERE_ON_CLOUDS
```

#### Option 3: Safer Sphere Intersection

```hlsl
// Common.ush line 1968
// In RayIntersectSphere()

// Replace:
if (Discriminant >= 0)

// With:
if (Discriminant >= -1e-6)  // Allow tiny negative for numerical stability
{
    Discriminant = max(0.0, Discriminant);  // Clamp before sqrt
```

#### Option 4: Use Radius Safety Margin

```hlsl
// SkyAtmosphere.usf around line 450
// In IntegrateSingleScatteredLuminance(), sphere intersection calls

// Add safety margin to sphere radius:
float2 SolB = RayIntersectSphere(WorldPos, WorldDir,
    float4(PlanetO, Atmosphere.BottomRadiusKm * PLANET_RADIUS_RATIO_SAFE_EDGE));
```

### C++ Fixes (SkyAtmosphereRendering.cpp)

#### Curvature-Aware Depth Test

```cpp
// After StartDepthViewCm calculation:
float ViewAltitudeKm = (ViewOrigin - PlanetCenter).Size() * CM_TO_KM - Atmosphere.BottomRadiusKm;
float CurvatureCorrection = 1.0f + FMath::Clamp(ViewAltitudeKm / Atmosphere.BottomRadiusKm, 0.0f, 0.5f);
StartDepthViewCm *= CurvatureCorrection;
```

#### Disable Depth Test Near Screen Edges

```cpp
// Pass edge distance to shader, disable depth test for edge pixels (~5% of screen)
PsPassParameters->ScreenEdgeDistance = ComputeScreenEdgeDistance(ViewRect);
// In shader: if (ScreenEdgeDistance < 0.05) skip depth test
```

### Option 5: Dynamic BottomRadius Based on Camera Altitude

A practical workaround: lerp the `BottomRadius` between ground-accurate and space-inflated values based on camera altitude. This avoids shader modifications entirely.

**Why it works:**
- At ground level: Use true mesh radius (no blue tint)
- At space level: Use slightly inflated radius (fixes edge artifacts)
- Smooth transition between the two

**Runtime cost:** Negligible - `SetBottomRadius()` uses state stream, just a uniform update (~microseconds/frame).

#### Implementation Options

**Option A: Blueprint (Recommended for prototyping)**

```
// Per-frame in PlayerController or CameraManager
Alpha = saturate((CameraAltitudeKm - GroundAltitude) / (SpaceAltitude - GroundAltitude))
BottomRadius = PlanetRadius + lerp(0.0, PlanetRadius * (InflationFactor - 1.0), Alpha)
SkyAtmosphereComponent->SetBottomRadius(BottomRadius)
```

**Option B: C++ Component**

```cpp
// Attach to camera pawn, override TickComponent
void UDynamicAtmosphereComponent::TickComponent(float DeltaTime, ...)
{
    const float AltitudeKm = (GetOwner()->GetActorLocation().Z * CM_TO_KM) - PlanetCenterZ;
    const float Alpha = FMath::Clamp((AltitudeKm - GroundAltitudeKm) / (SpaceAltitudeKm - GroundAltitudeKm), 0.f, 1.f);
    const float TargetRadius = PlanetRadiusKm * (1.0f + FMath::Lerp(0.f, InflationFactor, Alpha));

    // Smooth transition to avoid popping
    CurrentRadius = FMath::FInterpTo(CurrentRadius, TargetRadius, DeltaTime, SmoothSpeed);

    if (SkyAtmosphere)
        SkyAtmosphere->SetBottomRadius(CurrentRadius);
}
```

**Option C: Renderer-side (Engine modification)**

In `RenderSkyAtmosphere()` per-view loop:
```cpp
// Existing altitude calculation (from debug code)
const float AltitudeKm = (View.ViewLocation * CM_TO_KM - Atmosphere.PlanetCenterKm).Size() - Atmosphere.BottomRadiusKm;
const float Alpha = FMath::Clamp(AltitudeKm / TransitionAltitudeKm, 0.f, 1.f);
const float DynamicBottomRadiusKm = FMath::Lerp(Atmosphere.BottomRadiusKm, SpaceBottomRadiusKm, Alpha);
// Override in per-view uniform buffer
```

#### Recommended Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `GroundAltitude` | 0 km | Sea level |
| `SpaceAltitude` | 100 km | Full inflation threshold |
| `InflationFactor` | 0.005 (0.5%) | ~32 km at Earth scale |
| `SmoothSpeed` | 2.0 | Damping for transitions |

#### Lerp Formula

```
Alpha = saturate((CameraAltitudeKm - GroundAltitudeKm) / (SpaceAltitudeKm - GroundAltitudeKm))
BottomRadius = PlanetRadius * (1.0 + InflationFactor * Alpha)
```

- At 0 km: `Alpha=0` → `BottomRadius = PlanetRadius` (exact, no blue tint)
- At 100+ km: `Alpha=1` → `BottomRadius = PlanetRadius * 1.005` (fixes edge artifacts)

#### Gotchas

| Issue | Solution |
|-------|----------|
| Visual popping on rapid altitude change | Add damping: `FInterpTo()` with SmoothSpeed |
| Underground camera (Alt < 0) | Clamp Alpha to 0 |
| Multi-camera/split-screen | Compute per-viewport |
| VR stereo mismatch | Use head position, not per-eye |

#### Why NOT Shader-Only for Depth Code?

Using inflated radius ONLY in the depth buffer integration (lines 491-512) causes visual inconsistency:
- Sky gradient uses original radius
- Planet shadows use original radius
- Horizon scattering mismatches occlusion

**Recommendation:** Apply the lerp globally via `SetBottomRadius()` or not at all.

#### Alternative: Multiple SkyAtmosphere Actors

```
Alt < 10km:   GroundAtmosphereActor (precise radius, enabled)
10-100km:     TransitionAtmosphereActor (lerped)
100+km:       SpaceAtmosphereActor (inflated radius, enabled)
```

Toggle via `SetActorHiddenInGame()` with crossfade. Zero per-frame cost but requires careful blending.

### Recommended Fix Priority

1. **Option 5 (dynamic BottomRadius)** - No engine changes, works in Blueprint
2. **Option 2 (projected distance)** - Most robust single shader change
3. **Option 3 (safe sphere intersection)** - Defense in depth for all grazing rays
4. **Option 4 (radius margin)** - If edge artifacts persist, adds 10m safety
5. **Console vars** - Immediate workaround, no code changes needed

## Key Source Files

| File | Content |
|------|---------|
| `Engine/Source/Runtime/Engine/Private/Components/SkyAtmosphereComponent.cpp` | Component, scene proxy setup |
| `Engine/Source/Runtime/Renderer/Private/SkyAtmosphereRendering.cpp` | Renderer integration, LUT management, depth test setup |
| `Engine/Source/Runtime/Renderer/Private/SkyAtmosphereRendering.h` | Shader classes, render state |
| `Engine/Shaders/Private/SkyAtmosphere.usf` | Main HLSL shaders, ray marching, depth handling |
| `Engine/Shaders/Private/SkyAtmosphereCommon.ush` | Shared functions, medium sampling, constants |
| `Engine/Shaders/Private/Common.ush` | `RayIntersectSphere` implementation |

## Mobile Considerations

- SkyAtmosphere component needs a mesh tagged `IsSky` with a material using SkyAtmosphere nodes
- No real-time ray marching - samples pre-computed LUTs only
- Separate Mie/Rayleigh not available

## References

- [Sky Atmosphere Component - UE5 Docs](https://dev.epicgames.com/documentation/en-us/unreal-engine/sky-atmosphere-component-in-unreal-engine)
- [Sky Atmosphere Properties](https://dev.epicgames.com/documentation/en-us/unreal-engine/sky-atmosphere-component-properties-in-unreal-engine)
- [Cesium Ground-to-Space Fix](https://github.com/CesiumGS/cesium-unreal/issues/712)
- [SkyAtmosphere GitHub (Seb Hillaire)](https://github.com/sebh/UnrealEngineSkyAtmosphere)
- Example template: `Engine/Content/Maps/Templates/TimeOfDay_Default`
