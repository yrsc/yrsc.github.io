---
layout: post
title: "Unity热更新之AssetBundle的加载与卸载"
---

unity3d想要从一个AssetBundle里Load出资源，必须将该AssetBundle及该AssetBundle依赖到的其他AssetBundle都先加载到内存中，然后才能从该AssetBundle中Load资源。我们怎么知道一个AssetBundle依赖到了哪些其他的AssetBundle呢，这就需要提一下unity的打包机制。Unity3d调用打包函数`BuildPipeline.BuildAssetBundles`时，需要传进去一个Path,用于存放你打的AssetBundle，通常我们传进去的是`Application.streamingAssets`。然后在打包完成后，unity会默认生成一个和你存放AssetBundle的文件夹同名的assetbundle文件，用来存放所有AssetBundle的依赖关系，在这里，就会生成一个叫`StreamingAssets`的AssetBundle文件。因此，在加载某一个AssetBundle之前，我们都必须先加载这个名称叫做`StreamingAssets`的bundle文件，然后通过这个bundle文件寻找任意一个AssetBundle需要的依赖文件。

## 资源加载流程
举个例子：我们要加载prefab下的cube.prefab，加载流程如下：

```
1.LoadAssetbundle("StreamingAssets") //加载存放依赖关系的AssetBundle
2.LoadAssetbundle("prefab/cube.prefab")//加载目标预制件的AssetBundle
3.LoadDependcyAssetbundle("prefab/cube.prefab")//加载目标预制件依赖的AssetBundle
4.LoadAssetFromAssetBundle()//从cube.prefab这个AssetBundle里加载出资源
```
值得一提的是每一个AssetBundle在内存中只能有一份，重复加载同一个AssetBundle会报错，因此我们会需要一个字典来保存已经加载过AssetBundle。也就是说加载一个AssetBundle时，我们需要首先从字典中去取是否有该AssetBundle,如果有，直接从字典中取出，没有才需要从外部加载到内存中。

## 使用AssetBundle的加载资源方式和不使用AssetBundle加载资源的统一管理
快速开发阶段，我们往往希望在编辑器的模式下，不使用AssetBundle的方式加载资源，而是直接从编辑器下加载，但是打包出的apk或者ipa，我们又希望能够使用AssetBundle的方式去加载。
提供一种思路，可以用宏去区分是否使用AssetBundle的加载方式
```
public static T Load<T>(string path) where T : Object
{
	#if UNITY_EDITOR && !LOAD_ASSETBUNDLE_INEDITOR
		return UnityEditor.AssetDatabase.LoadAssetAtPath<T>(path);
	#else
		return AssetbundleLoader.LoadRes<T>(path);
	#endif
}
```
`LOAD_ASSETBUNDLE_INEDITOR`这个宏可以在buildsetting中去设置，去决定你是否想要在编辑器下使用AssetBundle的加载方式。
***另外，不建议把资源放在Resources目录中，然后通过`Resources.Load`的方式去加载。因为所有Resources目录下的文件都会默认打进apk或者ipa里，当你使用AssetBundle加载时，Resources目录下的文件没有任何用处，白白增加了包的大小。编辑器下建议通过`UnityEditor.AssetDatabase.LoadAssetAtPath`这种方式加载资源。***

文本文件不建议打包成AssetBundle更新，最好是直接下载更新，一个文本文件对应一个AssetBundle太浪费资源了，之前文章提过每一个AssetBundle需要4kb的内存。

## 不同平台下的AssetBundle默认存放路径

```
Android: Application.streamingAssetsPath + "!assets/AssetBundle/";
iOS: Application.dataPath + "/Raw/AssetBundle/";
Editor: Application.dataPath + "/StreamingAssets/AssetBundle/";
```
我们根据需要加载的资源名称去对应平台下的路径寻找AssetBundle即可。如果考虑到热更新的情况，我们需要先去AssetBundle的下载目录查看是否存在该AssetBundle,如果不存在，再去默认路径下加载该AssetBundle。

## 资源的卸载
AssetBundle有2种卸载方式，`AssetBundle.unload(true)`，`AssetBundle.unload(false)`。详情看<http://www.cnblogs.com/88999660/archive/2013/03/15/2961663.html>。
实际应用中，我们很难去判断什么时候需要去调用unload(false),稍微使用不当就会造成资源的多份拷贝。我的理解是assetbundle打的越少越好，这样即使不调用unload(false)也不会有太多内存的开销，参见上一篇"打包AssetBundle"。如果有强迫症，一定要卸载，最好在场景切换的时候去unload(false)上一个场景的AssetBundle。至于texture，mesh的卸载，在切换场景或者内存峰值的时候调用`Resources.UnloadUnusedAssets()`来卸载已经没有引用的资源即可。

工程地址参见：<https://github.com/yrsc/AssetBundleFramework>