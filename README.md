# Water Caustics Shader

Water Caustics Shader implemented with Shader Graph in **Unity 2021.3.10f1**

## Screenshots

### Table of Content

- [Implementation](#implementation)
  - [Scene Setup](#scene-setup)
  - [HLSL](#hlsl)
    - [Main Light](#main-light)
    - [UV Rotation](#uv-rotation)
    - [Triplanar Projection](#triplanar-projection)

### Resources

- [Custom Lighting in Shader Graph](https://blog.unity.com/technology/custom-lighting-in-shader-graph-expanding-your-graphs-in-2019)
- [Caustics Texture](https://graphicdesign.stackexchange.com/questions/4725/how-can-i-create-a-large-size-water-caustics-texture-for-animation)
- [Stylized Skybox](https://assetstore.unity.com/packages/2d/textures-materials/sky/free-stylized-skybox-212257)

#### HLSL

- [Triplanar ShaderGraph Node HLSL code](https://docs.unity3d.com/Packages/com.unity.shadergraph@12.1/manual/Triplanar-Node.html)
- [UV Rotation ShaderGraph Node HLSL code](https://docs.unity3d.com/Packages/com.unity.shadergraph@12.1/manual/Rotate-Node.html)

## Implementation

### Scene Setup

- Built a basic pool and put some primitive objects inside.
- Used the stlylized skybox to make the scene look more polished.

![Picture](./docs/1.jpg)

### HLSL

#### Main Light

- Based on the example for implementing [custom lighting in Shader Graph](https://blog.unity.com/technology/custom-lighting-in-shader-graph-expanding-your-graphs-in-2019), we define the custom HLSL function to get the main light.
- `SHADERGRAPH_PREVIEW` will be true while developing the shader in the Shader Graph node editor.

```c
void MainLight_half(
    in float3 WorldPos,
    out half3 Direction,
    out half3 Color,
    out half DistanceAtten,
    out half ShadowAtten
)
{
    #if SHADERGRAPH_PREVIEW
        Direction = half3(0.5, 0.5, 0);
        Color = 1;
        DistanceAtten = 1;
        ShadowAtten = 1;
    #else
        #if SHADOWS_SCREEN
            half4 clipPos = TransformWorldToHClip(WorldPos);
            half4 shadowCoord = ComputeScreenPos(clipPos);
        #else
            half4 shadowCoord = TransformWorldToShadowCoord(WorldPos);
        #endif

        Light mainLight = GetMainLight(shadowCoord);
        Direction = mainLight.direction;
        Color = mainLight.color;
        DistanceAtten = mainLight.distanceAttenuation;
        ShadowAtten = mainLight.shadowAttenuation;
    #endif
}
```

#### UV Rotation

- Simply a copy paste of the HLSL code from the documentation for the [Shader Graph Node to do UV roation](https://docs.unity3d.com/Packages/com.unity.shadergraph@12.1/manual/Rotate-Node.html).

```c
float2 Unity_Rotate_Degrees_float(float2 UV, float2 Center, float Rotation)
{
    Rotation = Rotation * (3.1415926f/180.0f);
    UV -= Center;

    float s = sin(Rotation);
    float c = cos(Rotation);

    float2x2 rMatrix = float2x2(c, -s, s, c);

    rMatrix *= 0.5;
    rMatrix += 0.5;
    rMatrix = rMatrix * 2 - 1;

    UV.xy = mul(UV.xy, rMatrix);
    UV += Center;

    return UV;
}
```

#### Triplanar Projection

- Based on the HLSL code from the [Shader Graph Node to do Triplanar Projection](https://docs.unity3d.com/Packages/com.unity.shadergraph@12.1/manual/Triplanar-Node.html).
- The modifications for this include passing in a **Speed** and **Rotation** for the UVs.
- The **offset** will be calculated with **Time** and **Speed**.
- The **rotation** will be used with the previously defined HLSL function.
- This returns a **projection** based on the **world position** of the **vertices**, using each **plane** as an **UV mapping**.
  - This means the **projection** will always be the same, no matter the shape or position of the object.
  - There will be a **blending** done using the Normals orientation and a blend parameter.

```c
void TriplanarProjection_float(
    in Texture2D Texture,
    in SamplerState Sampler,
    in float3 Position,         // world space
    in float3 Normal,           // world space
    in float Tile,
    in float Blend,

    // UV manipulation
    in float Speed,
    in float Rotation,

    out float4 Out
)
{
    float3 Node_UV = Position * Tile;

    // animate UVs
    float Offset_UV = _Time.y * Speed;

    float3 Node_Blend = pow(abs(Normal), Blend);
    Node_Blend /= dot(Node_Blend, 1.0);

    float4 Node_X = SAMPLE_TEXTURE2D(Texture, Sampler, Unity_Rotate_Degrees_float(Node_UV.zy, 0, Rotation) + Offset_UV);
    float4 Node_Y = SAMPLE_TEXTURE2D(Texture, Sampler, Unity_Rotate_Degrees_float(Node_UV.xz, 0, Rotation) + Offset_UV);
    float4 Node_Z = SAMPLE_TEXTURE2D(Texture, Sampler, Unity_Rotate_Degrees_float(Node_UV.xy, 0, Rotation) + Offset_UV);

    Out = Node_X * Node_Blend.x + Node_Y * Node_Blend.y + Node_Z * Node_Blend.z;
}
```
