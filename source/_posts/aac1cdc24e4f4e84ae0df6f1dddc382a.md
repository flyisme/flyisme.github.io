---
layout: post
title: LaunchMode
abbrlink: aac1cdc24e4f4e84ae0df6f1dddc382a
tags: []
categories:
  - Mac笔记本
  - Android
  - 四大组件
date: 1718265554873
updated: 1718726082869
---

Activity `LaunchMode`

## 任务栈

为什么要有任务栈?

- 应用A 添加联系人:调用通信录应用的添加联系人页面,点击返回按钮应该返回添加联系人页面,且 按 task按键 不应该显示联系人 应用.与打开它的task相关.
- 编写邮箱:应用A 发邮件: 按Task按键: 应该显示应用A 与邮箱

前台TASK 按下返回键会逐级返回.

### 默认规则:

在不同Task中打开同一个Activity,Activity会被创建多个实例,分别放进每一个 Task 中.

### Task 由前台进入后台

1. Home按键返回桌面
2. 最近任务键 查看最近任务
   1. 显示最近 任务时 就进入后台了

### `android:allowTaskReparenting="true"`

> Android 9,Android 10 失效了
> ![aeacecffb4d09bf4ac3633ad1144ac29.png](/resources/64bc0b9e21d44afd972735437e37c846.png)
> 短信应用打开 "发送邮件(清单文件里申明了该属性)" 页面 , 点击Home键后,在桌面打开 邮件 会发现 '发送邮件'页面位于栈顶,后续在打开 短信应用,则无发送邮件也页面了.即:发送邮件页面 任务栈移动到了 邮件中.

点击桌面任务栈，看到的是 task，切换app，实际上切换的是task

## taskAffinity

默认是包名

- 默认情况下,Acitivity会直接进入当前Task

- 但对于设置了 launcherMode="singleTask" 的Activity,系统会比对Acitivity和当前TASK 的TASKAffinity 是否相同:
  - 相同入栈
  - 不同,查找 和它的taskAffinity后入栈
  - 找不到,创建一个新Task

- Standard
  - 默认启动模式：场景：社区业务：查看用户A信息->查看用户A粉丝->查看用户A 粉丝的用户信息...

- SingleTop
  - 栈顶复用：`onNewIntent()` ：通知栏进入的场景。

- SingleTask
  - 栈内复用不会存在多个实例：`onNewIntent()` 应用首页。
  - 被别的App 启动时,放在自己的栈顶,不在别的App task
  - 应用间的启动动画(Task 间)
  - 清除TASK 前的 Activity
  - 保证了一个TASK里有这个Acitivity,又保证了这个TASK里只有这个Activity. 全局只有一个对象

- SingleInstance
  - 加强的SingleTask模式：这个TASK里有且只能有一个Activity：系统应用。

### Intent.FLAG\_ACTIVITY\_NEW\_TASK分析

对于非Activity启动的Activity(比如Service或者通知中启动的Activity),需要显式设置`FLAG_ACTIVITY_NEW_TASK`

ContextImpl startActivity:

```java
@Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity"
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
```

### mTaskIdToAffinity

```java
// 在 ActivityTaskManagerService 中
private val mTaskIdToAffinity = HashMap<Int, String>()

fun startActivity(..., taskAffinity: String? = null) {
    // ...
    val taskId = if (taskAffinity != null) {
        findTaskIdByAffinity(taskAffinity) ?: createNewTask(taskAffinity)
    } else {
        // 使用应用包名作为 Affinity
        val appPackageName = ...
        findTaskIdByAffinity(appPackageName) ?: createNewTask(appPackageName)
    }
    // ...
}

private fun findTaskIdByAffinity(affinity: String): Int? {
    return mTaskIdToAffinity.entries.find { it.value == affinity }?.key
}

private fun createNewTask(affinity: String): Int {
    val newTaskId = generateNewTaskId()
    mTaskIdToAffinity[newTaskId] = affinity
    return newTaskId
}
```

映射过程如下:

1. 启动Activity: 当启动一个新的Activity时,系统会检查该Activity 是否具有taksAffinity属性
2. 查找任务栈
   - 如果Acitivity具有taskAffinity属性,ATMS会根据该属性在mTaskIdToAffinity中查找对应的TaskId.
   - 如果查找到对应的TaskId,说明存在一个具有相同Affinity的任务栈,将该Acitivity添加到任务站中
   - 如果未找到TaskId,说明不存在具有相同Affinity的任务栈,则创建一个新的任务栈,并将该Acitivity添加到该任务栈中,同时将新的TaskID和Affinity映射关系添加到mTaskToAffinity中
3. 无taskAffinity 属性:如果Acitivity没有taskAffinity 属性,则默认使用应用包名作为Affinity,并按照上述步骤查找或创建任务栈.

疑问❓默认的taskaffinity source的?
