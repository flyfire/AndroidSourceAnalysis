![](http://ww1.sinaimg.cn/large/006tKfTcly1ffh81xy9abj30yg0q00wu.jpg)
![](http://ww1.sinaimg.cn/large/006tKfTcly1ffh82hjymdj30pg0me402.jpg)
最简单的Activity启动方式, 如下:
```
Intent intent = new Intent(this, TestActivity.class);
startActivity(intent);
```
startActivity方法存在于Activity类中, Activity类的调用流程:
+ ``Activity#startActivity(Intent)``
+ ``Activity#startActivityForResult(Intent, int, Bundle)``
Activity#startActivityForResult核心是mInstrumentation.execStartActivity
![](http://ww2.sinaimg.cn/large/006tKfTcly1ffh8gqipnwj31hg108wqx.jpg)
![](http://ww4.sinaimg.cn/large/006tKfTcly1ffh8hpc7p2j31kw0qzk0j.jpg)
![](http://ww4.sinaimg.cn/large/006tKfTcly1ffh8j59lk5j319u0de432.jpg)
execStartActivity核心是ActivityManagerNative类的getDefault#startActivity, 获取IActivityManager单例, 本质是ServiceManager.getService("activity"), 即ActivityManagerService. 通过Binder, 远程调用ActivityManagerService类的startActivity方法.Activity启动通过远程调用, 由当前应用交给ActivityManagerService(AMS)处理.

ActivityManagerService类是负责启动Activity的核心服务, 简称AMS. 启动逻辑包含在ActivityStarter, ActivityStackSupervisor和ActivityStack和三个类中, ActivityStarter负责启动, ActivityStackSupervisor负责管理Stack和TaskRecord, ActivityStack负责管理栈内的Activity.

Stack和TaskRecord示例:

```
Stack #1:
    Running activities (most recent first):
      TaskRecord{3caa65e3 #2711 A=me.chunyu.spike.wcl_activity_launchmode_demo U=0 sz=2}
        Run #1: ActivityRecord{36b06e99 u0 me.chunyu.spike.wcl_activity_launchmode_demo/.TestAActivity t2711}
        Run #0: ActivityRecord{27396226 u0 me.chunyu.spike.wcl_activity_launchmode_demo/.MainActivity t2711}
  Stack #0:
    Running activities (most recent first):
      TaskRecord{27d796c9 #2695 A=com.miui.home U=0 sz=1}
        Run #0: ActivityRecord{2e5712cb u0 com.miui.home/.launcher.Launcher t2695}
```

ActivityManagerService类的调用过程:
+ ``ActivityManagerService#startActivity``
+ ``ActivityManagerService#startActivityAsUser``
ActivityManagerService调用ActivityStarter的startActivityMayWait方法, 执行启动.
![](http://ww2.sinaimg.cn/large/006tKfTcly1ffh8msjjh7j31kw0d944h.jpg)
ActivityStarter类负责处理Activity的Intent和Flags的逻辑, 还有管理Stack和TaskRecord.ActivityStarter类的调用流程:

+ ``ActivityStarter#startActivityMayWait``
+ ``ActivityStarter#startActivityLocked``
+ ``ActivityStarter#startActivityUnchecked``

**startActivityMayWait**: 根据Intent获取Activity的启动信息(ResolveInfo和ActivityInfo), 获取调用者的Pid和Uid.
**startActivityLocked**: 创建ActivityRecord, 含有Activity的核心信息.
**startActivityUnchecked**: 根据启动的Flag信息, 设置TaskRecord, 完成后执行ActivityStackSupervisor类的resumeFocusedStackTopActivityLocked方法, 继续启动.

```
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
    // ...
    // 所有启动准备完成后, dontStart是true.
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || mLaunchSingleTop || mLaunchSingleTask);
    if (dontStart) {
        ActivityStack.logStartActivity(AM_NEW_INTENT, top, top.task);
        // doResume是true, mDoResume也是true 
        if (mDoResume) {
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }
        // ...
    }
    // ...
}
```
ActivityStackSupervisor类与ActivityStack类配合使用. ActivityStackSupervisor负责管理Task和Stack, 而ActivityStack负责管理在Stack和Task中的Activity. 因此, 对于Stack和Task的操作, AMS使用ActivityStackSupervisor进行管理; 对于Activity的操作, AMS使用ActivityStack进行管理. 两者相互调用, 最终完成启动Activity.

ActivityStackSupervisor类与ActivityStack类的调用流程:
+ ``ActivityStackSupervisor#resumeFocusedStackTopActivityLocked``
+ ``ActivityStack#resumeTopActivityUncheckedLocked``
+ ``ActivityStack#resumeTopActivityInnerLocked``
+ ``ActivityStackSupervisor#startSpecificActivityLocked``

ActivityStackSupervisor类的startSpecificActivityLocked方法调用realStartActivityLocked方法, 执行真正的启动Activity.在ActivityStackSupervisor的realStartActivityLocked方法中, 含有启动的核心方法scheduleLaunchActivity, 即调用IApplicationThread的scheduleLaunchActivity方法.IApplicationThread的实现是ApplicationThread, 而ApplicationThread是ActivityThread的内部类, 即使用ApplicationThread类的scheduleLaunchActivity方法处理Activity启动.最终由ActivityThread完成Activity的创建与绘制.

ActivityThread#handleLaunchActivity:handleLaunchActivity调用performLaunchActivity方法, 继续执行启动, 在成功后, 调用handleResumeActivity方法, 执行显示Activity.
![](http://ww3.sinaimg.cn/large/006tKfTcly1ffhn8c4z5tj31kw0kfjyg.jpg)
ActivityThread#performLaunchActivity:performLaunchActivity方法是Activity启动的核心:
+ 获取Activity的组件信息(ComponentName);
+ 根据组件使用反射创建Activity(newActivity);
+ 将Activity绑定(attach)Application和BaseContext;
+ 相继调用onCreate, onStart, onRestoreInstanceState, onPostCreate等方法.
+ 放入Activity列表(Map)中统一管理, token是key.

通过分析performLaunchActivity, 我们也更加清晰Activity的生命周期, 顺序如下, onCreate, onStart, onRestoreInstanceState, onPostCreate. 注意, onStart是Activity处理; 其余三个是Instrumentation处理, 支持继承重写相应方法, 自行处理.

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```


