#Android 开发锁屏屏蔽Home键、原生锁屏界面
最近开发锁屏应用，需要屏蔽返回、音量、Home键，返回和音量很好弄，关键是Home键如何搞。花了很多心思，终于搞定了，项目代码涉密，不方便全传上来，下面主要介绍这个思路。
## 一、原理
Android2.3版本以前通过AttachWindows就可以很简单实现屏蔽，下面着重说4.0以后的。
访谈及结果分析
### 1、向系统注册锁屏界面Activity为启动器
这块是需要将该应用设定为桌面程序：

    <intent-filter >
      <category android:name="android.intent.category.HOME" />
    </intent-filter>

### 2、设置该应用为默认启动器

多启动器共存的情况下当HOME事件发生系统会轮询并向用户询问使用哪个启动器并是否作为默认启动器，这一部分不可控，可做为用户引导，让用户选择**本应用**为默认启动器

### 3、选择真正的系统桌面

第2步骤中若用户选择的是其它桌面作为默认启动器，则无法屏蔽HOME键，此为用户体验问题了。可以在APP中做提醒引导用户去执行第2步操作来实现**本应用**作为默认启动器；

当第2步中**本应用**被选作默认启动器，则系统HOME事件将从此会导向**本应用**做处理：

    注意：此处应该通过getIntent().getCategories()来获取是否是来自与Home事件的启动项。
    android.intent.category.LAUNCHER：点击图标启动
    android.intent.category.HOME：点击Home启动
因为**本应用**不具备桌面管理功能，故第一次HOME事件发生时**本应用**应该拦截HOME事件，轮询系统桌面管理应用提供给用户选择（得到的list会包含**本应用**，应remove掉，剩下的给用户选择），记下用户的选择，这样，在非锁屏状态时的HOME事件来到星酷时应当跳转到用户选择的桌面上去，完成“回到桌面”功能，而当锁屏状态，**本应用**正处于最前台进程，可忽略HOME事件，这就完成了“屏蔽HOME”功能了。

    判断锁屏是否为前台方法为，判断是否在内存堆栈的栈顶：
    ActivityManager manager=(ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
    String name = manager.getRunningTasks(1).get(0).topActivity.getClassName();
    name.equals(LockActivity.class.getName())  
----
    

    轮询系统启动器／桌面管理应用方法为:
    Intent mIntent = new Intent(Intent.ACTION_MAIN);
    mIntent.addCategory(Intent.CATEGORY_HOME);
    packageManager.queryIntentActivities(mIntent, 0);

### 4、屏蔽系统原生锁屏界面

首先需要动态监听系统的开／关屏事件（注意是动态注册监听，不能在manifest中固定注册，否则无法实现），再则事件发生时启动**本应用**的锁屏Activity，在setContentView之前执行
getWindow().addFlags(WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD| WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED);


### 5、锁屏状态下电话事件处理

通话过程中由于距离传感器的原因，会造成屏幕关闭，所以也会触发锁屏，这样便无法进行操作。因此通话过程中应当禁止启动锁屏界面。处理方法如下：

    TelephonyManager phoneManager = (TelephonyManager) this.getSystemService(TELEPHONY_SERVICE);
		// 手动注册对PhoneStateListener中的listen_call_state状态进行监听
		phoneManager.listen(new PhoneStateListener() {
			@Override
			public void onCallStateChanged(int state, String incomingNumber) {
				switch (state) {
				case TelephonyManager.CALL_STATE_IDLE:
					break;
				case TelephonyManager.CALL_STATE_RINGING:
					finish();
					break;
				case TelephonyManager.CALL_STATE_OFFHOOK:
					finish();
				default:
					break;
				}
				super.onCallStateChanged(state, incomingNumber);
			}
		}, PhoneStateListener.LISTEN_CALL_STATE);


### 6、阻止锁屏界面出现在长按Home键的“最近使用应用列表”中

    在manifest中对锁屏Application加 android:excludeFromRecents="true"
----
    
    为避免某些异常，最好在锁屏的Activity添加 
    android:noHistory="true"
    android:launchMode="singleTask"
    

### 7、项目需要的权限
    <uses-permission android:name="android.permission.DISABLE_KEYGUARD" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
    <uses-permission android:name="android.permission.PROCESS_OUTGOING_CALLS" />

#### 综上，一个锁屏界面需要的操作流程大致如下：

* listenAndHandleScreenOffAction();
* listenAndHandlePhoneAction();
* dismissKeyguard();
* handleHomeIntent();
* processViewEvents();
* handleUnlockEvent();

## 二、关键代码
暂时无法提供
