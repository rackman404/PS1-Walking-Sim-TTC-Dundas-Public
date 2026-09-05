Small modification to increase from base 4 vertex lights to engine max of 8 vertex lights.

# Modification Reasoning
Original purpose was to allow more leeway for lighting my large contiguous enviroment meshes. Mostly unneeded now since i've readded in Unity lightmapping feature to the shader again

# Implementation

Simple modification, increased a parameter from 4 to 8, this function is called elsewhere and would now sample the closest 8 lights instead of 4

PSX-ShaderSrc.cginc
```c
#ifdef PSX_ENABLE_CUSTOM_VERTEX_LIGHTING
	return ShadePSXVertexLightsFull(vertex, normal, 8, true); //changing 4 to 8 so theres more leeway with my environment meshes
#else
	return ShadeUnityVertexLightsFull(vertex, normal, 8, true);
#endif
```