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

For opaque geometry, the ray marcher reads the depth buffer to limit ray distance. This is where edge artifacts originate:

```hlsl
// SkyAtmosphere.usf lines 491-512 (UE 5.6)
if (DeviceZ != FarDepthValue)
{
    const float3 DepthBufferTranslatedWorldPosKm = GetScreenTranslatedWorldPos(SVPos, DeviceZ).xyz * CM_TO_SKY_UNIT;
    const float3 TraceStartTranslatedWorldPosKm = WorldPos + View.SkyPlanetTranslatedWorldCenterAndViewHeight.xyz * CM_TO_SKY_UNIT;
    const float3 TraceStartToSurfaceWorldKm = DepthBufferTranslatedWorldPosKm - TraceStartTranslatedWorldPosKm;
    float tDepth = length(TraceStartToSurfaceWorldKm);  // Issue 1: length() includes perpendicular error
    if (tDepth < tMax)
    {
        tMax = tDepth;
    }

    if (dot(WorldDir, TraceStartToSurfaceWorldKm) < 0.0)  // Issue 2: No epsilon tolerance
    {
        return Result;  // Early exit with zero luminance - causes dark edges
    }
}
```

### Ray-Sphere Intersection

The `RayIntersectSphere` function in `Common.ush` uses standard quadratic formula:

```hlsl
float Discriminant = QuadraticCoef.y * QuadraticCoef.y - 4 * QuadraticCoef.x * QuadraticCoef.z;
if (Discriminant >= 0)  // No epsilon - vulnerable at grazing angles
{
    float SqrtDiscriminant = sqrt(Discriminant);
    Intersections = (-QuadraticCoef.y + float2(-1, 1) * SqrtDiscriminant) / (2 * QuadraticCoef.x);
}
```

**Precision constants defined but underutilized:**
- `PLANET_RADIUS_OFFSET = 0.001` (1 meter in km) - defined but not used in sphere intersection calls
- `PLANET_RADIUS_RATIO_SAFE_EDGE = 1.00000155763` (~10m at Earth scale) - used for atmosphere entry, not depth tests

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

| Factor | Location | Issue |
|--------|----------|-------|
| **Dot check - no epsilon** | SkyAtmosphere.usf:509 | Strict `< 0.0` fails on FP noise at grazing angles |
| **Length vs projected distance** | SkyAtmosphere.usf:496 | `length()` includes perpendicular error; `dot()` would be robust |
| **Ray-sphere discriminant** | Common.ush:1968 | `>= 0` check with no epsilon for near-zero cases |
| **Radius offset unused** | SkyAtmosphereCommon.ush:26 | `PLANET_RADIUS_OFFSET` defined but not used in sphere tests |
| **Planar depth test** | SkyAtmosphereRendering.cpp | `StartDepthZ` computed from center ray, no curvature compensation |
| **Inverted Z precision** | SkyAtmosphere.usf:~450 | Known issue: "bad values when DeviceZ is far=0" |

**Epic's own TODOs acknowledge FP issues:**
```hlsl
// Line ~450: TODO: investigate why SvPositionToWorld returns bad values when DeviceZ is far=0
// Line 1529: TODO: investigate why we need this workaround... floating point issue with sphere intersection?
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
// Replace:
if (dot(WorldDir, TraceStartToSurfaceWorldKm) < 0.0)

// With:
if (dot(WorldDir, TraceStartToSurfaceWorldKm) < -0.001)  // 1 meter tolerance in km
```

#### Option 2: Recommended - Use Projected Distance

Fixes both length precision and dot check in one change:

```hlsl
// SkyAtmosphere.usf lines 491-515 - Full replacement for the #else block:
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

In `Common.ush` around line 1968:

```hlsl
// Replace:
if (Discriminant >= 0)

// With:
if (Discriminant >= -1e-6)  // Allow tiny negative for numerical stability
{
    Discriminant = max(0.0, Discriminant);  // Clamp before sqrt
```

#### Option 4: Use Radius Safety Margin

In sphere intersection calls within `IntegrateSingleScatteredLuminance`:

```hlsl
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

### Recommended Fix Priority

1. **Option 2 (projected distance)** - Most robust single shader change
2. **Option 3 (safe sphere intersection)** - Defense in depth for all grazing rays
3. **Option 4 (radius margin)** - If edge artifacts persist, adds 10m safety
4. **Console vars** - Immediate workaround, no code changes needed

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
