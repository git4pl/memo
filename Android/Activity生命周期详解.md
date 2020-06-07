### 1. Activity正常启动：
onCreate -> onStart -> onResume

### 2. Activity启动另一个Activity：
（1）B完全遮挡住A
A:onPause -> B:onCreate -> B:onStart -> B:onResume -> A:onSaveInstanceState-> A:onStop
此时如果点击Back键，将依次执行B:onPause -> A:onRestart -> A:onStart -> A:onResume -> B:onStop -> B:onDestroy，但是如果A长时间在后台被杀死，则会调用其onCreate

（2）B没有完全遮挡住A（比如Dialog形式或半透明）
A:onPause -> B:onCreate -> B:onStart -> B:onResume -> A:onSaveInstanceState
此时如果点击Back键，将依次执行B:onPause ->  A:onResume -> B:onStop -> B:onDestroy
此时如果点击home键，将依次执行B:onPause ->  A:onStop -> B:onSaveInstanceState -> B:onStop
再将应用切到前台 A:onRestart -> A:onStart -> B:onRestart -> B:onStart -> B:onResume
以上讨论的基础都是在B Activity的启动模式为standard模式下，如果B Activity的启动模式为singleTask或singleInstance， 而且B Activity 已经存在于栈中，则启动B Activity时，不会调用其onCreate，而是调用onNewIntent。

### 3. 按back键返回到桌面（或者调用finish()方法销毁）
onPause -> onStop -> onDestroy

### 4. 按home键返回到桌面
onPause -> onSaveInstanceState -> onStop

### 5.Activity 上有 Dialog 的时候按 home 键返回到桌面，之后再返回到Activity
弹出Dialog不影响Activity的生命周期（），点击home键后onPause ->   onSaveInstanceState ->  onStop
再将Activity切换到前台onRestart ->  onStart -> onResume
需要注意和dialog形式的Activity区分，弹出的AlertDialog对话框实际上是Activity的一个组件，我们对Activity并不是不可见而是被一个布满屏幕的组件覆盖掉了其他组件，所以我们无法对其他内容进行操作，也就是AlertDialog实际上是一个布满全屏的组件，因此不会影响当前Activity的生命周期。但是启动一个dialog形式的Activity情况就不同了，参见第2条。

### 6. 锁屏再开屏
onPause -> onSaveInstanceState -> onStop  ->  onRestart ->  onStart -> onResume

### 7.旋转屏幕
onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume

### 8.下拉状态栏
不影响Activity生命周期
