## 1. Fragment

### 1.1 生命流程

onAttach -> onCreate -> onCreateView -> onActivityCreated -> onStart -> onResume -> onPause -> onStop -> onDestoryView -> onDestory -> onDetach

### 1.2 Fragment 与Activity交互

- 接口回调
- 将数据放到Bundle通过setArguments传入Fragment
- 接口回调
- getActivity得到Activity类然后调用方法
- 广播

### 1.3 add replace

add时会保留之前fragment状态

replace不会保留之前fragment状态

## 2. 触摸事件

### 2.1 触摸事件传递

Activity 接受触摸事件 -> ViewGroup(dispatchTouchEvent、onInterceptTouchEvent) -> View(OnTouch(声明了OnTouchListener)、OnTouchEvent)

OnTouchEvent返回true代表事件被消费，则不需要再上传，返回false表示事件没有被消费，逐级上报，直到Activity里的OnTouchEvent()方法消费事件

onTouch方法返回True时OnTouchEvent就不会执行

### 2.2 Down Move Up

在Down事件之后OnTouchEvent返回true，之后的Move Up都会到这个View中

如果Down事件之后OnTouchEvent返回false，之后的Move Up都不会到这个View，直到下一个down事件判断

### 2.3 onInterceptTouchEvent

- 拦截Down 事件的分发
- 中止Up 和Move事件向目标View 传递，使得目标View所在的ViewGroup捕获Up和Move事件

## 3. Android启动模式

- Stander：每次启动Activity之后都在task栈顶新建
- SingleTop：**栈顶复用模式**  如果启动的Activity在栈顶则复用，否则新建
  - 当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调**onNewIntent**方法
  - 当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例
  - 当前栈中不存在该Activity的实例时，其行为同standard启动模式
- SingleTask：**栈内复用模式** 如果task栈中存在，则将上面的其他Activity移除，然后复用，否则新建然后放入栈顶
- SingleInstance：**全局唯一模式 **即只要有任何一个栈存在此Activity实例，就会复用此实例，回调onNewIntent方法。如果此实例不存在，那么就会创建新的Task栈，并放入Activity实例。 （只要Activity存在，就存在与单独的一个栈中）