
# Showcase
<img width="800" height="450" alt="Image" src="https://github.com/user-attachments/assets/ef2c1ccc-643a-4763-9640-246284e818f8" />

# Shader Effect/Requirements
- Based on a given world position (in this case, player position) the shader will change transparency (essentially providing the player a means of seeing that its the game border when it gets close)
- Should accept a colored and textured border

# Implementation Reasoning
- Basic shader practice (only really need to do shit in fragment shader)
- Allow for graphically showing the player the bounds of the current playable area (without just having a invisible boundary)

# External Data
- Player position (provided in BoundaryShaderManager.cs)
	- Updated on every frame using a global vector (float4)
```c#
    void UpdateValues()
    {   
        Shader.SetGlobalVector("_Boundary_PlayerPos", new Vector4(transform.position.x, transform.position.y, transform.position.z, 0));
    }
```


# Internal Data
The shader will accept the following parameters for customization:
- Basic RGB
- Texturing (with alphas)
- an "intensity" range (the "max" clamp value)
- Range where player will start seeing the effect

```c

    Properties {
        _Color("Color (RGBA)", Color) = (1, 1, 1, 1)
        _MainTex ("Base (RGB) Trans (A)", 2D) = "white" {}
        _Boundary_EffectMaxIntensity ("Max Effect Intensity", Float) = 1
        _Boundary_EffectRange ("Effect Range", Float) = 0.8
    }
```

# Implementation
- No separate .cginc used, shader is simple enough to just do it in line tbh.

In order for the effect to work, we must calculate the position of the  given pixel in a fragment shader with respect to both its own position, and the player position. We must therefore pass this information from vertex -> fragment data struct
```c
	//https://discussions.unity.com/t/getting-the-world-position-of-the-pixel-in-the-fragment-shader/177100/2
	struct v2f
	{
		float2 uv : TEXCOORD0;
		float4 vertex : SV_POSITION;
		float3 worldPos: TEXCOORD7;
	};
```
This was done by just using a unused texture coord channel (in this case the last one) to be reserved for storing the current pixel world coord


```c

	v2f vert (appdata v)
	{
		v2f o;
		o.vertex = UnityObjectToClipPos(v.vertex);
		o.uv = TRANSFORM_TEX(v.uv, _MainTex);
		//UNITY_TRANSFER_FOG(o,o.vertex);

		//we need world pos at pixel to compare to player pos
		o.worldPos = mul(unity_ObjectToWorld, v.vertex);

		return o;
	}
```
One modification was done to the default vertex function, retransforming the vertex position back into world space. We note that the 'o' struct will be automatically interpolated in the fragment function from nearby vertexes (which will give the correct pixel coord from the vertex coord calculated here).

```c
	//use global vector4 defined in boundary
	fixed3 _Boundary_PlayerPos;
	float _Boundary_EffectRange;
	float _Boundary_EffectMaxIntensity;
	/*
	main part of shader
	2 modes:
	(textured mode): preapplied texture + alpha changing 
	- 
	(no textures mode): no textures, will use solid color + alpha changing
	- for things like a checkerboard pattern texture (with preexisting)
	*/
	fixed4 frag (v2f i) : SV_Target
	{ 
		//sample the texture
		fixed4 col = tex2D(_MainTex, i.uv); 

		//make transparent
		col.a = 0;
		
		//NOTE, use a keyword instead or someshit for the mat?
		if (length(_Color.rgb) != length(float3(1,1,1))){ //if user actually set a custom color 
			col += fixed4(_Color.r,_Color.g,_Color.b,clamp(1-distance(i.worldPos,_Boundary_PlayerPos)/_Boundary_EffectRange, 0, _Boundary_EffectMaxIntensity)); 
		}//else default to red
		else{
			col += fixed4(1,0,0,clamp(1-distance(i.worldPos,_Boundary_PlayerPos)/_Boundary_EffectRange, 0, _Boundary_EffectMaxIntensity)); 
		}

		return col;
	}
```
this is kinda ass but can be optimized later,

Boundary.Shader


