

For some reason the original dev for the PSX shader kit never actually integrated unity's baked lighting into her shaders. As a result, i've re added basic baked lighting into the shaders (though whether it's actually close to how baked lighting is implemented on the PSX remains to be seen, have to research more)

# Modification Reasoning
- Technically can just bake the data directly into color maps in a external program (i.e maya/blender) but this
- From a technical perspective 

# Shaders Modified
Of the various shaders in this kit, only "vertexlit" was modified (normal and lite versions), reason being that stuff like transparent versions should support baking lighting into the textures in the first place

### Implementation

Snippet of "PSX-VertexLit.Shader"
``` c
		//Vertex lighting requires two passes to be defined. One used with lightmaps and one without.
        Pass
        {
            Tags { "LightMode" = "VertexLM" }
            CGPROGRAM
            #pragma vertex vert
            #pragma geometry geom
            #pragma fragment frag
            #pragma multi_compile_fog
            #pragma multi_compile_geometry __ PSX_ENABLE_CUSTOM_VERTEX_LIGHTING
            #pragma multi_compile_geometry __ PSX_FLAT_SHADING_MODE_CENTER
            #pragma multi_compile PSX_TRIANGLE_SORT_OFF PSX_TRIANGLE_SORT_CENTER_Z PSX_TRIANGLE_SORT_CLOSEST_Z PSX_TRIANGLE_SORT_CENTER_VIEWDIST PSX_TRIANGLE_SORT_CLOSEST_VIEWDIST PSX_TRIANGLE_SORT_CUSTOM
            
            //https://discussions.unity.com/t/using-ifdef-lightmap_on-or-ifndef-lightmap_off/478308/8 allows for checking if baked lighting was generated
            #pragma multi_compile _ LIGHTMAP_ON

            #include "UnityCG.cginc"
            #include "PSX-Utils.cginc"

			samplerCUBE _Cubemap;
            sampler2D _ReflectionMap;
			float4 _CubemapColor;
			
            #define PSX_VERTEX_LIT
			#define PSX_CUBEMAP _Cubemap
			#define PSX_CUBEMAP_COLOR _CubemapColor
			
            #include "PSX-ShaderSrc.cginc"

        ENDCG
        }
```

We note that a pass for when lightmaps are generated is already included in shader, only modifications here was  the "\#pragma multi_compile _ LIGHTMAP_ON" which adds the LIGHTMAP_ON keyword for detecting when to actually enable the lightmap code later on in the .cginc code

Geometry -> Fragment data struct (PSX-ShaderSrc.cginc)
```c
struct g2f
{
	float4 affineUV1 : TEXCOORD0;
	float4 vertex : SV_POSITION;
	float customDepth : TEXCOORD1;

#ifdef PSX_VERTEX_LIT
	float4 color : COLOR0;
#endif 
	UNITY_FOG_COORDS(2)
#ifdef PSX_VERTEX_LIT
	float4 affineUV2 : TEXCOORD3;
#endif
#ifdef PSX_CUBEMAP
	float3 reflectionDir : TEXCOORD4;
#endif

#ifdef PSX_VERTEX_LIT
	#ifdef LIGHTMAP_ON 
		float2 lightuv : TEXCOORD7; 
	#endif
#endif
};
```
One extra texture coordinate was added onto the struct called "lightuv" (mapped to TEXCOORD7 as TEXCOORD2 was used elsewhere I believe). This is needed because we can't directly map the lightmaps to the other UVs (or at least i wasn't able to). 

Geometry Fragment Program (PSX-ShaderSrc.cginc)
```c
[maxvertexcount(3)]
void geom(triangle v2g IN[3], inout TriangleStream<g2f> triStream)
{
	float4x4 matrix_mv = UNITY_MATRIX_MV;
	float4x4 matrix_p = UNITY_MATRIX_P;
	
#ifndef PSX_TRIANGLE_SORT_OFF
	float triSortDepth = PSX_TRIANGLE_SORTING_FUNC(IN[0].vertex, IN[1].vertex, IN[2].vertex);
#else
	float triSortDepth = 0;
#endif

	// First pass to prepare data for all the triangles to potentially use later
	g2f o[3];
	for (int i = 0; i < 3; i++)
	{
		o[i].vertex = mul(matrix_mv, IN[i].vertex);
#ifdef PSX_VERTEX_LIT
		fixed3 viewNormal = normalize(mul((float3x3)UNITY_MATRIX_IT_MV, IN[i].normal));
		o[i].color.rgb = ShadePSXVertexLights(o[i].vertex, viewNormal);
		o[i].color.a = 1; 
#endif

/* cba to deal with for the other shader version, should just only be used on vertex_lit and if there are lightmaps baked in the scene */
#ifdef LIGHTMAP_ON
	//modified from https://github.com/TwoTailsGames/Unity-Built-in-Shaders/blob/master/DefaultResourcesExtra/Normal-VertexLit.shader
	o[i].lightuv = IN[i].uv.xy * unity_LightmapST.xy + unity_LightmapST.zw;  //calculate lightmap position(idfk what is happening here)
	//(x,y,1,0) (where (x,y) is the actual UV coords, where 1 to let affine factor be unchanged, 0 because original uses a vector size of 4 with last element unused (idfk why))
#endif
o[i].affineUV1 = CalculateAffineUV(o[i].vertex, TRANSFORM_TEX(IN[i].uv, _MainTex)); 

//.... REST OF GEOMETRY FRAGMENT
```
The only changes needed was to map the Unity generated lightmaps in the geo program was to map the lightmap UVs to the actual UVs of this object.


```c
#ifndef PSX_TRIANGLE_SORT_OFF
fragOut frag(g2f i, UNITY_VPOS_TYPE screenPos : SV_POSITION)
#else
fixed4 frag(g2f i, UNITY_VPOS_TYPE screenPos : SV_POSITION) : COLOR
#endif
{
	fragOut o;

	//if baking was done AND shader is vertex lit, use the lightmap
	#ifdef PSX_VERTEX_LIT
		#ifdef LIGHTMAP_ON 
			fixed4 lightTex = UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightuv.xy); //sample the light map given the uv positions calculated in vertex shader
			half4 bakedColor = half4(DecodeLightmap(lightTex), 1.0); //(r,g,b,a)? (decodeLightmap itself only decodes to a half3 data type, a might not be used at all
			//fixed4 colorTex = tex2D(_MainTex, i.nonAffineUV.xy);
			//o.color = fixed4(1,1,1,1); //test

			//For this pixel in the fragment shader, bake the brightness to it
			o.color = bakedColor; 
			//apply the base color property if applied in inspector
			o.color *= _Color;

			//sample the color from the texture given the affine UV calculated in vertex shader
			fixed4 colorTex = tex2D(_MainTex, i.affineUV1.xy / i.affineUV1.z);
			//multiply the color by the brightness from the baked light map
			o.color.rgb = colorTex.rgb * o.color.rgb;
		#else
			o.color = tex2D(_MainTex, i.affineUV1.xy / i.affineUV1.z) * _Color; //original
		#endif
	#else
		o.color = tex2D(_MainTex, i.affineUV1.xy / i.affineUV1.z) * _Color; //original
	#endif
	
//... REST OF FRAGMENT CODE HERE
```
Here is where the main code for dealing with baked lighting happen. 

In terms of execution flow:
- Applying baked lights should only happen if A. there are actual lightmaps in the scene and B. if the shader currently run is explicitly the "VertexLit" one. I have wrapped this in 2 different IFDEF statements as im not sure if you can use a an statement or smth. 
- In addition I have two ELSE statements as idk how to use just a single one

In terms of actually applying the baked lighting:
- We essentially apply the "brightness" from the baked lighting texture set that unity has setup first
	- This is done by sampling the lightmap texture W.R.T to the UVs defined and calculated from the geometry program
	- From there we use a prebuilt "DecodeLightmap" function built into a Unity cginc included elsewhere, this should decode the data Into the proper format as needed
	- Lastly we simply copy the brightness into the pixel
	- We also apply the \_Color property on top of this (if I have set one in inspector, otherwise this does nothing obv.)
		- Technically we can just do this further below I think, but the actual source code does it here so fuck it

Example of above, in this case, we get this view (if we omit all further code beyond applying the baked lighting from the code snippet above, we can clearly that we should have applied the baked lighting properly) (note AO applied around the base of pillars and darker lighting on corners and edges)
![[Pasted image 20260826142117.png]]

Next we simply apply the actual color map texture atop of the baked lighting:

```c
	fixed4 colorTex = tex2D(_MainTex, i.affineUV1.xy / i.affineUV1.z);
```
The above simply does PS1 style texture warping (as done originally in this shader)
``` c
	o.color.rgb = colorTex.rgb * o.color.rgb;
```
The above now multiplies the brightness already recorded into the pixel from above to the actual color of the pixel.

The final result, we note that by multiplying the baked lights data with the color map data we essentially darken/brightness the given pixel as needed
![[Pasted image 20260826142417.png]]


### Lite Version
Exact same implementation as with the normal version


# Further notes on how Lightmaps work



![[Pasted image 20260826123707.png]]