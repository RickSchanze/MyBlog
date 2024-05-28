---
title: UGUI源码阅读(1)--从Button开始
tags:
  - UGUI
  - Unity
categories:
  - UGUI
typora-root-url: ./..
abbrlink: 5f2caef2
date: 2024-05-13 22:39:45
---

# 前言

本篇是阅读UGUI源码系列的第一篇，阅读他人的源码是提升自己的快速途径之一，考虑到之后去往实习时UI方面的工作应该会比较多，因此阅读一下UGUI源码对于工作应该会有很大的帮助。

通过阅读本篇，可以了解到：Button点击事件的执行流程，点击时如何检测到相应的UI元素等。

注意：**本文采用的Unity版本是Unity6000.0.0f1c1**

下一篇：[UGUI源码阅读(2)--Image的更新流程 | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/79fdc967.html)

# 开始

首先，我们编写一个脚本，用于Debug。

```C#
public class UGUITest : MonoBehaviour
{
    [SerializeField] private Button _testButton;

    public void OnButtonClick()
    {
        Debug.Log("Button Clicked!");
    }
}
```

正确的设置后，在唯一的`Debug.Log()`语句处打上断点，开始调试！![image-20240526151044398](/imgs/post/UGUI源码阅读-1-从Button开始/image-20240526151044398.png)

可以看到，点击事件调用栈从`EventSystem.Update()`开始。

加入我们删除Hierarachy中的"EventSystem"对象，可以发现：按钮无法被点击，但是如果在代码中写入如下代码：

```C#
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Mouse0))
        {
            Debug.Log("K");
        }
    }
```

会发现，控制台会正常输出“K”，这说明**EventSystem是专门用于处理UI的输入的一个组件**。

点进EventSystem类，我们先从EventSystem的成员看起，这里仅列出与点击事件执行有关的成员：

```C#
    public class EventSystem : UIBehaviour
    {
        // 当前所有已注册的EventSystem，会在Update里调用TickModules
        private List<BaseInputModule> m_SystemInputModules = new List<BaseInputModule>();
        // 当前的InputModule，调试时是一个InputSystemUIInputModule
        private BaseInputModule m_CurrentInputModule;
        // 所有的EventSystem
        private static List<EventSystem> m_EventSystems = new List<EventSystem>();
        /// <summary>
        /// Return the current EventSystem.
        /// </summary>
        public static EventSystem current
        {
            get { return m_EventSystems.Count > 0 ? m_EventSystems[0] : null; }
            set
            {
                int index = m_EventSystems.IndexOf(value);

                if (index > 0)
                {
                    m_EventSystems.RemoveAt(index);
                    m_EventSystems.Insert(0, value);
                }
                else if (index < 0)
                {
                    Debug.LogError("Failed setting EventSystem.current to unknown EventSystem " + value);
                }
            }
        }
    }
```

对于一个正在运行的场景，拥有多个EventSystem是不被允许的，会在Update中给予警告。

# EventSystem的执行流程

在EventSystem的`Update`中，发生了如下的事：

## TickModule

```C#
if (current != this)
    return;
TickModules();
```

```c#
private void TickModules()
{
    var systemInputModulesCount = m_SystemInputModules.Count;
    for (var i = 0; i < systemInputModulesCount; i++)
    {
         if (m_SystemInputModules[i] != null)
              m_SystemInputModules[i].UpdateModule();
         }
    }
}
```

在每一帧中首先会确保Tick的是当前的组件，之后对所有存储的InputModule进行Tick。

`InputSystemUIModule`并没有重写`UpdateModule()`函数。

## 验证、切换当前生效的InputModule

```C#
bool changedModule = false;
var systemInputModulesCount = m_SystemInputModules.Count;
// 检查当前InputModule是否被切换
for (var i = 0; i < systemInputModulesCount; i++)
{
    var module = m_SystemInputModules[i];
    if (module.IsModuleSupported() && module.ShouldActivateModule())
    {
        if (m_CurrentInputModule != module)
        {
            // 切换InputModule
            ChangeEventModule(module);
            changedModule = true;
        }
        break;
    }
}
```

上面的代码首先查看当前激活的module是不是已经激活的module，如果不是的话则会进行调用`ChangeEventModule`进行切换，这个函数里会调用`BaseInputModule`中有关切换的回调：

```C#
private void ChangeEventModule(BaseInputModule module)
{
    if (m_CurrentInputModule == module)
        return;

    if (m_CurrentInputModule != null)
        m_CurrentInputModule.DeactivateModule();

    if (module != null)
        module.ActivateModule();
    m_CurrentInputModule = module;
}
```

假如此时当前的InputModule没有设置的话，就会进入设置流程：

```c#
// no event module set... set the first valid one...
if (m_CurrentInputModule == null)
{
    for (var i = 0; i < systemInputModulesCount; i++)
    {
        var module = m_SystemInputModules[i];
        if (module.IsModuleSupported())
        {
            ChangeEventModule(module);
            changedModule = true;
            break;
        }
    }
}
```

而只有当没有切换InputModule以及当前InputModule已经被有效设置时才会进行事件处理：

``` C#
if (!changedModule && m_CurrentInputModule != null)
    m_CurrentInputModule.Process();
```

经过试验，假如场景中没有有效的InputModule，那么将无法触发Button的点击事件。

## 事件处理

事件处理主要在`Process()`函数，其中我们暂时不关注导航输入（导航输入），因此下面直接看鼠标处理的部分。

```C#
public override void Process()
{
    // 处理过期的Pointer
    if (m_NeedToPurgeStalePointers)
        PurgeStalePointers();

    // Reset devices of changes since we don't want to spool up changes once we gain focus.
    if (!eventSystem.isFocused && !shouldIgnoreFocus)
    {
        for (var i = 0; i < m_PointerStates.length; ++i)
            m_PointerStates[i].OnFrameFinished();
    }
    else
    {
        // 处理NavigationInput
        // ...
        // Pointer input.
        for (var i = 0; i < m_PointerStates.length; i++)
        {
            ref var state = ref GetPointerStateForIndex(i);

            ProcessPointer(ref state);

            // If it's a touch and the touch has ended, release the pointer state.
            // NOTE: We defer this by one frame such that OnPointerUp happens in the frame of release
            //       and OnPointerExit happens one frame later. This is so that IsPointerOverGameObject()
            //       stays true for the touch in the frame of release (see UI_TouchPointersAreKeptForOneFrameAfterRelease).
            if (state.pointerType == UIPointerType.Touch && !state.leftButton.isPressed && !state.leftButton.wasReleasedThisFrame)
            {
                RemovePointerAtIndex(i);
                --i;
                continue;
            }

            state.OnFrameFinished();
        }
    }
}
```

处理鼠标输入时，首先获取了对应索引的鼠标状态，即`GetPointerStateForIndex`，此函数获取`InputSystemUIInputModule`内部存储的`InlineArray<PointerModel>`类型的`m_PointerStates`对应索引的成员并返回。

而对于结构`PointerModel`，这是一个保存了鼠标事件数据的结构，其中我们主要使用其中`eventData`成员。

接下来进入`ProcessPointer(ref state)`函数：

它首先会根据鼠标指针的类型和鼠标本身的状态(`None`、`Locked`等)来对`PointerModel`中`eventData`的数据进行更新，在我们这里简单的鼠标点击情况下，会触发最后一个else分支：

```C#
else
{
    eventData.delta = state.screenPosition - eventData.position;
    eventData.position = state.screenPosition;
}
```

修改了鼠标位置与其Delta。

之后的关键操作便是射线检测我们触碰到的UI元素。

```c#
// ...
// Raycast from current position.
eventData.pointerCurrentRaycast = PerformRaycast(eventData);
```

在我们这个简单的例子中RaycastResult中的`m_GameObject`成员是Button下面的TextMeshPro组件。

接下来会以此处理鼠标移动、鼠标左键事件、鼠标拖拽事件，鼠标右键事件和拖拽事件以及中键相关事件，这里我们主要关注鼠标左键的事件：

```C#
// ...
ProcessPointerButton(ref state.leftButton, eventData);
// ...
```

在`ProcessPointerButton`函数中会首先处理“按下”事件，然后处理“释放”事件，而在处理“释放”事件时会同时处理“Click”事件，处理这些事件的逻辑流程区别不大，我们主要看释放事件：

首先获取当前射线检测到的UI对象：

```C#
var currentOverGo = eventData.pointerCurrentRaycast.gameObject;
```

然后在此对象上寻找实现了`IPointerClickHandler`的对象：

```C#
var pointerClickHandler = ExecuteEvents.GetEventHandler<IPointerClickHandler>(currentOverGo);
```

`ExecuteEvents`是一个静态类，包含一系列帮助事件处理的功能，比如上面的函数会从当前的对象开始向上寻找第一个实现了`IPointerClickHander`的对象，这里我们就寻找了`Button`组件。

随后我们会依次调用鼠标释放`PointerUp`、`PointerClick`等事件，对于PointerClick，它是如此调用的：

```C#
if (isClick)
    // 这里传入了ExecuteEvents.pointerClickHandler，是事先准备好的静态委托，它会调用IPointerClickHandler的OnPointerClick方法。
    ExecuteEvents.Execute(eventData.pointerPress, eventData, ExecuteEvents.pointerClickHandler);
```

上面这个函数真正调用了Click事件函数：

```C#
public static bool Execute<T>(GameObject target, BaseEventData eventData, EventFunction<T> functor) where T : IEventSystemHandler
{
    var internalHandlers = ListPool<IEventSystemHandler>.Get();
    GetEventList<T>(target, internalHandlers);
    var internalHandlersCount = internalHandlers.Count;
    for (var i = 0; i < internalHandlersCount; i++)
    {
        T arg;
        try
        {
            arg = (T)internalHandlers[i];
        }
        catch (Exception e)
        {
            var temp = internalHandlers[i];
            Debug.LogException(new Exception(string.Format("Type {0} expected {1} received.", typeof(T).Name, temp.GetType().Name), e));
            continue;
        }

        try
        {
            functor(arg, eventData);
        }
        catch (Exception e)
        {
            Debug.LogException(e);
        }
    }

    var handlerCount = internalHandlers.Count;
    ListPool<IEventSystemHandler>.Release(internalHandlers);
    return handlerCount > 0;
}
```

它会尝试获取所有实现了`IPointerClickHander`的函数并进行调用。在关键函数`functor(arg, eventData);`中会调用：

```C#
private static void Execute(IPointerClickHandler handler, BaseEventData eventData)
{
    handler.OnPointerClick(ValidateEventData<PointerEventData>(eventData));
}
```

进而调用`Button`的`Process()`函数，此函数调用`m_OnClick.Invoke();`从而调用到我们自己写的Button回调，这便是整个事件处理的流程。

# 总结

本文介绍了Unity UGUI的一个`Button`从点击到处理我们自己写的回调函数所经历的所有代码阶段。
