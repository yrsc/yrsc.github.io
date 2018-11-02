---
layout: post
title: scriptObject创建模板的用法
tag: 编辑器
---

scriptObject可以用来存储游戏数据，以此建立一个模板，然后调整不同的参数值，就可以生成多种不同的物品了。

GunTemplate 继承于ScriptObject

```C#
using UnityEngine;
using UnityEditor;
using System.IO;

public class GunGenerator
{
    [MenuItem("Tools/GenerateGun")]
    public static void GenerateGun()
    {
        var templatePath = "Data/Gun/template.asset";
        var assetPath = Application.dataPath + "/" + templatePath;
        if(File.Exists(assetPath))
        {
            //选中之前创建的模板
            EditorUtility.FocusProjectWindow();
            Selection.activeObject = AssetDatabase.LoadMainAssetAtPath("Assets/" + templatePath);
            return;
        }
        var gunData = ScriptableObject.CreateInstance<GunTemplate>();
        AssetDatabase.CreateAsset(gunData, "Assets/Data/Gun/template.asset");
        AssetDatabase.SaveAssets();
        //选中刚刚创建的模板
        EditorUtility.FocusProjectWindow();
        Selection.activeObject = gunData;
    }
}


```