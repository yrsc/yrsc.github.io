---
layout: post
title: 利用Projector实现伪动态阴影
tag: 渲染
---
传统的阴影实现方法是基于ShadowMap的原理，ShadowMap是以光源为视角，先渲染一张深度图缓存起来，记录下物体在光源空间的深度，然后在实际的渲染过程中，将接受阴影的物体的转换到光源空间之后的深度与深度图作比较，如果比深度图要深，则该点处于阴影之中。

Unity自带的阴影在某些手机上存在硬件不支持的情况，另一种实现阴影的方式可以通过Projector+RenderTexture来实现。我们通过在projector上挂载阴影相机，将相机渲染到RenderTexture上，再将RenderTexture作为贴图传递给Projector的材质，即可实现动态阴影。

projector上挂载的脚本如下:
```
[RequireComponent(typeof(Projector))]
public class ShadowCameraGenerator : MonoBehaviour 
{
	public LayerMask shadowCasterLayer;
	private RenderTexture _renderTex;

	void Awake()
	{
		Projector projector = GetComponent<Projector>();
		Camera camera = gameObject.AddComponent<Camera>();
		camera.cullingMask = shadowCasterLayer;
		camera.orthographic = true;
		camera.orthographicSize = projector.orthographicSize;
		camera.clearFlags = CameraClearFlags.SolidColor;
		camera.backgroundColor = new Color(1.0f,1.0f,1.0f,0.0f);
		_renderTex = new RenderTexture(1024,1024,0,RenderTextureFormat.ARGB32);
		camera.targetTexture = _renderTex;
		camera.SetReplacementShader(Shader.Find("Unlit/ReplaceShadowRender"),null);
		projector.material.SetTexture("_ShadowTex",_renderTex);
	}

	//ignore black quad occur when editor stop play
	#if UNITY_EDITOR
	void OnApplicationQuit()
	{	
		Projector projector = GetComponent<Projector>();
		projector.material.SetTexture("_ShadowTex",null);
	}
	#endif
}
```
使用Projector来实现阴影时，需要注意的几点：
1、传递给Projector材质的RenderTexture必须是clamp模式，但是如果阴影到了RenderTexture边缘的像素，因为是Clamp的原因，地板就会出现整个长条形的阴影，解决方案可以通过给projector材质添加mask图来处理边缘的像素。
2、使用阴影相机时，设置replaceshader,因为只需要物体的alpha信息，因此阴影相机渲染时使用最简单的shader即可。
3、RenderTexture的大小会决定阴影的质量，图越小，阴影质量越差。

Projector Shader
```
Shader "Projector/Multiply" {
	Properties {
		_ShadowTex ("Cookie", 2D) = "black"{}
		_FalloffTex ("FallOff", 2D) = "white" {}
	}
	Subshader {
		Tags {"Queue"="Transparent"}
		Pass {
			ZWrite Off
			ColorMask RGB
			Blend DstColor Zero
			Offset -1, -1

			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#pragma multi_compile_fog
			#include "UnityCG.cginc"
			
			struct v2f {
				float4 uvShadow : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 pos : SV_POSITION;
			};
			
			float4x4 unity_Projector;

			v2f vert (float4 vertex : POSITION)
			{
				v2f o;
				o.pos = mul (UNITY_MATRIX_MVP, vertex);
				o.uvShadow = mul (unity_Projector, vertex);
				UNITY_TRANSFER_FOG(o,o.pos);
				return o;
			}
			
			sampler2D _ShadowTex;
			sampler2D _FalloffTex;
			
			fixed4 frag (v2f i) : SV_Target
			{
				fixed4 texS = tex2Dproj (_ShadowTex, UNITY_PROJ_COORD(i.uvShadow));
				fixed4 texF = tex2Dproj (_FalloffTex, UNITY_PROJ_COORD(i.uvShadow));
				fixed ratio =  texF.r * texS.a;
				fixed4 res = fixed4(1,1,1,1) *  (1 - ratio);
				UNITY_APPLY_FOG_COLOR(i.fogCoord, res, fixed4(1,1,1,1));
				return res;
			}
			ENDCG
		}
	}
}

```
工程地址参见： <a href="https://github.com/yrsc/Shadow" target="_blank">https://github.com/yrsc/Shadow</a>