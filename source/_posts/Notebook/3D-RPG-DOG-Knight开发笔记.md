---
title: 【3D-RPG】DOG-Knight开发笔记
index_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-DirectX-12-3D-游戏开发实战-笔记.png
banner_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-DirectX-12-3D-游戏开发实战-笔记.png
layout: post
categories: [Notebook]
tags: [Unity]
sticky: 100
---

unity开发

<!-- more -->

## 前言

本项目参考 [教程](https://www.bilibili.com/video/BV1rf4y1k7vE?spm_id_from=333.999.0.0&vd_source=1614cbf0a650cda8c72c85046d0d9e51)开发。B站上有免费的核心功能教程（36课），付费的教程需要购买。

开发本项目的在于学习unity使用。

前置教程：

- Unity官方 C#初级编程 [点击跳转](https://www.bilibili.com/video/BV1oy4y1q7jJ)
- Unity官方 C#中级编程 [点击跳转](https://www.bilibili.com/video/BV1f5411G7bp)

## 一 准备工作

开发环境：unity  2020.3.26f1c1 LTS 

IDE：VSCode

创建3D项目，自行升级到URP渲染管线：

- 安装URP插件，右键创建Pipeline Asset，放入Project Settings->Graphics/Quality中的可编程渲染管线设置。
- 在素材商店选择合适的免费素材导入到工程中，可以点击素材中的.unitypackage文件升级到URP，也可以在Edit->Render Pipeline中升级。

## 二 搭建场景

Window->Rendering->Lighting 设置并烘焙灯光。

Shadows在URP Asset里设置。

**视点移动**：

- alt+左键  第三人称转动   
- 中键 平移    
- 右键 第一人称转动
- 右键+WASD 第一人称视角漫游

**放置技巧**：

- v选择顶点，按住拖拽自动吸附到其它顶点上；ctrl+shift，按住拖拽自动吸附到其它平面上

- ctrl+shift+F：快速定位相机到当前窗口

- scene面板中F或hierarchy窗口中双击：快速定位到选中物体

安装的第三方工具可以通过Tools使用。Polybrush调整地形，快速刷上Prefab。Probuilder快速制作素材。ProGrids参考线辅助移动。

使用Navigation烘焙智能导航

cinemachine添加跟随人物的相机（CM FreeLook）。

创建Global Volume生成各种效果（记得在Main Camera里面启动Rendering->Post Processing）。

为人物自制Animator，不同Layer放置不同类别动画，在代码中Set Parameters来控制动画。可以通过Animator Override Controller基于现有Animator快速创建新的。可以为每个Animation添加单独的Behaviour脚本。

使用Shader Graph自制材质。

在URP Asset自带的Forward Render中添加Render Objects，为指定的Layer的游戏对象设置遮挡剔除（09课）。

想让游戏对象不阻挡射线，可以设置为Ignore Raycast Layer或者关闭Mesh Collider。



## 三 代码编写

### 3.1 Managers

此类代码用于全局管理，都是单例的。单例泛型类实现如下.

此外，它们不随关卡变化销毁，需在各自```Awake()```方法里加上```DontDestroyOnLoad(this);```。

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Singleton<T> : MonoBehaviour where T : Singleton<T>
{
    private static T instance;

    public static T Instance
    {
        get => instance;
    }

    public static bool IsInitialized
    {
        get => instance != null;
    }


    protected virtual void Awake()
    {
        if (instance != null)
            Destroy(gameObject);
        else
            instance = (T)this;
    }

    protected virtual void OnDestory()
    {
        if (instance == this)
        {
            instance = null;
        }
    }


}
```

#### 3.1.1 MouseManager

**通过观察者模式控制鼠标点击触发的操作**

继承`UnityEngine.Events`中的`UnityEvent<>`可以建立事件类。（由于没有继承自MonoBehaviour，需要给类加上[System.Serializable]，其变量才能在窗口中显示出来。）在窗口中控制事件类的对象触发的操作。

也可以自行创建委托代替事件类，编写代码控制委托触发的操作。这里我是用这种方法。

**Update()**

`Camera.main.ScreenPointToRay(Input.mousePosition)`获取摄像机向鼠标位置发出的光线，`Physics.Raycast(ray, out hitInfo)`获取光线碰撞信息到hitInfo，然后根据hitInfo获取游戏对象的tag，根据tag设置鼠标的纹理`Cursor.SetCursor`。

`Input.GetMouseButtonDown(0)`当鼠标左键点击时，根据当前点击游戏对象的tag触发委托，进行移动、攻击等操作。

#### 3.1.2 GameManager

创建IEndGameObserver接口作为观察者：

```c#
public interface IEndGameObserver
{
    void EndNotify();
}
```

使用列表`List<IEndGameObserver>`储存观察者，主体包含一下管理观察者的方法：

```c#
public void AddObserver(IEndGameObserver observer)
{
    endGameObservers.Add(observer);
}

public void RemoveObserver(IEndGameObserver observer)
{
    endGameObservers.Remove(observer);
}

public void NotifyObservers()
{
    foreach (var observer in endGameObservers)
    {
        observer.EndNotify();
    }
}
```

此外在GameManager中添加玩家状态、相机这些需要全局使用的游戏对象。

#### 3.1.3 SaveManager

将游戏数据用JsonUtility转化为json，利用PlayerPrefs保存游戏数据。Windows的默认保存位置在注册表HKEY_CURRENT_USER\SOFTWARE\Unity\UnityEditor\公司名\产品名。

**Update()**

根据键盘输入进行保存和加载操作。

### 3.2 Character

此类代码用于游戏主角/敌人的控制。仅简单描述其逻辑。

#### 3.2.1 PlayerController

**Awake()**

通过`GetComponent<>`获取身上的各种组件。

**OnEnable()**

将指定的方法添加到MouseManager的委托中，使其被鼠标点击触发。

将玩家的CharacterStats注册到GameManager中。

**Start()**

加载存档数据。

**OnDisable()**

为了防止场景切换时发生错误，将指定的方法从MouseManager的委托中移除。

**Update()**

如果死亡，通过GameManager向所有观察者发布死亡消息。

通过Animator切换/控制Animation。

记录攻击CD的倒计时。

**移动**

中断所有协程。控制NavMeshAgent。

**攻击**

获取攻击目标，进行暴击判定。通过协程移动并攻击。

`Physics.OverlapSphere(transform.position, sightRadius)`找到视野内的所有colliders。

使用Animation Event控制伤害造成的时机。对可击飞目标的Rigidbody加力。

#### 3.2.2 EnemyController

通过`[RequireComponent(typeof(NavMeshAgent))]`自动添加组件。

通过`[Header("Basic Settings")]`设置窗口头部。

使用枚举EnemyStates规定敌人的状态`GUARD,PATROL,CHASE,DEAD`。

实现IEndGameObserver接口（以作为观察者）的`EndNotify()`方法，进行被通知之后的操作。

创建ExtensionMethod为transform添加`IsFacingTarget(this Transform transform, Transform target)`扩展方法，便于判断角色是否面对目标

**Awake()**

通过`GetComponent<>`获取身上的各种组件。

**Start()**

通过GameManager添加自己为观察者。

**OnDisable()**

移除观察者。

**Update()**

根据当前状态行动（switch）。切换动画。记录攻击CD的倒计时。

使用`Random.Range`和`NavMesh.SamplePosition`获得随机巡逻点。

**OnDrawGizmosSelected()**

绘制显示敌人视野范围的小组件。

**子类Grunt**

为Grunt添加击飞方法

**子类Golem**

为Golem添加击飞方法和扔石头方法

### 3.3 Character Stats

此类代码用于记录游戏数据。

CharacterData_SO、AttackData_SO继承自ScriptableObject，使用字段记录基本数据、攻击数据。CharacterStats类使用属性和方法提供对前者的控制。

### 3.4 UI

使用canvas、image、button。

#### 3.4.1 HealthBarUI

挂载在每个敌人身上。需要在敌人身上创建barPoint作为血条位置参考。

**Awake()**

将血条更新方法添加到CharacterStats的委托中，在收到伤害时触发。

 **OnEnable()**

对于Render Mode为World Space的HealthBar Canvas，每个脚本均创建独立的Bar Holder，以红/绿image叠加显示血条。

**LateUpdate()**

更新血条位置（根据敌人位置）和朝向（根据相机朝向）

刷新血条显示冷却时间。

#### 3.4.2 PlayerHealthUI

挂载在玩家身上。Render Mode为Screen Space。在每一帧更新血量和经验值。

#### 3.4.3 MainMenu

挂载在Menu Canvas上。给主界面UI相应按钮注册鼠标点击事件。

#### 3.4.4 SceneFader

挂载在Fade Canvas上。利用Canvas Group的alpha通道实现渐入渐出。

### 3.5 Transition

#### 3.5.1 TransitionDesitination

目的地枚举`ENTER, A, B, C, EXIT`。

#### 3.5.2 TransitionPoint

传送类型枚举`SameScene, DifferentScene`。

传送门挂载TransitionPoint和 TransitionDesitination，设定传送类型和目的地。

将传送门的Box Collider标记为Is Trigger，通过`void OnTriggerStay(Collider other)`和`void OnTriggerExit(Collider other)`方法改变变量canTrans，标记是否可以传送。

**Update()**

按E传送。

#### 3.5.3 SceneController

单例模式。

通过协程异步加载场景并传送。

所有的传送方法均放在这里。

## 四 打包及运行

File->Build Settings中将所有场景加入，Edit->Project Settings->Player中设置游戏名。

使用IL2CPP作为脚本后端打包，游戏更小性能更高。

## 版本控制

版本控制系统**plasticSCM**有两种使用方式：

1. 通过unity hub创建仓库把项目托管到plasticSCM，使用unity自带的plastichub作为服务器 [Plastic Hub (unity.cn)](https://plastichub.unity.cn/)。

2. 在引擎里打开Window里的plasticSCM，通过uniy登录plasticSCM官网，创建仓库，使用官方 [Plastic SCM](https://www.plasticscm.com/)作为服务器。

以上两种方式建立的仓库的服务器不同，但在登录同一个plasticSCM账号的客户端都能看到。后者这个有更大的免费容量（50GB）。

如果已经安装了plasticSCM客户端，此时不再需要安装plasticSCM插件。不然会出现编译错误（dll冲突）和奇怪BUG（Preference里的External Tools选项卡为空白）。
