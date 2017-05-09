从setContentView开始看
**AppCompatActivity.setContentView**
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffdprrttqhj311205gaba.jpg)
**AppCompatActivity.getDelegate**
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffdptzk9b2j312m084dhd.jpg)
**AppCompatDelegate.create**
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffdpvgmf4kj31a00hwtep.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffdpwip835j30ua090ju6.jpg)
**AppCompatDelegateImplV9.setContentView**
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffdpyqn4fij31j809mwhm.jpg)   
在``ensureSubDecor``中主要调用了``createSubDecor``
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffdq7jeduyj31fs03wjs9.jpg)
在``createSubDecor``中调用了``mWindow.getDecorView``
**PhoneWindow.getDecorView**
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffdq9iro42j30xm08u0u1.jpg)
**PhoneWindow.installDecor**
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffdqbrsqyvj31ey0godke.jpg)
**PhoneWindow.generateDecor**
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffdqd9yp2mj31kk0m0jxe.jpg)
返回了一个``DecorView``,``DecorView``实际上是一个``FrameLayout``
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffdqf2khkrj31kg034my6.jpg)
再来看``PhoneWindow.generateLayout(DecorView decor)``方法
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffev9pbr9bj310004wwfn.jpg)
**PhoneWindow.getWindowStyle**
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffevb2vfspj318w0f6adc.jpg)
**PhoneWindow.generateLayout(DecorView decor)**中根据window theme中各种不同的attributes来request feature, 然后根据feature来决定要inflate得layout布局id.
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffevffs0u3j31ac0dugpv.jpg)
**DecorView.onResourcesLoaded(LayoutInflater inflater, int layoutResource)**
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffevgtov3yj31js0u612b.jpg)
可以看到把上步得到的layout布局给inflate了并且添加到了DecorView中，也就是FrameLayout中。不妨看一下上一步得到的布局``screen_simple``是什么吧。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffevrs1qb3j319s0kcn5d.jpg)
ViewStub延迟加载的``action_mode_bar``layout。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffevtnd1xzj30zy07840d.jpg)
我们再返回到**AppCompatDelegateImplV9.createSubDecor**中``mWindow.getDecorView``之后。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffevwpyww7j31fc0hgdkp.jpg)
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffevxtawhoj31ek07ytb1.jpg)
可以看到也是根据window不同的特征inflate不同的layout.看下``abc_screen_simple``的布局。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffew047cdtj314m0k6dm8.jpg)
``abc_screen_content_include``布局
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffew15pivaj318m0bogp1.jpg)
subDecor inflate得到布局之后。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffew6w11ofj31kw0sstii.jpg)
``mWindow.findViewById``实际上就是``getDecorView.findViewById``
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffew7zpdxfj30sm05ct9o.jpg)
注意``windowContentView.setId(View.NO_ID);``把DecorView中id为``android.R.id.content``的id设置为了``NO_ID``.同时``contentView.setId(android.R.id.content);``把subDecor中的内容id设置为了``android.R.id.content``，这主要是为了兼容早期版本。
在``mWindow.setContentView(subDecor)``中
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffewcw356dj31kw0usdpn.jpg)
可以看到类似于DecorView中layout为screen_simple的布局中id为``android:id/content``的FrameLayout将subDecor infalte出的布局给addView了，同时进行了``android.R.id.content``的替换。
终于看完了DecorView的部分，我们再返回到``AppCompatDelegateImplV9.setContentView(int resId)``上面来，也就是最初调用的地方。
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffdpyqn4fij31j809mwhm.jpg) 
可以看到我们自定义的布局，比如``activity_main``被inflate并添加到了subDecor布局中id为``android.R.id.content``的ContentFrameLayout中。至此，完成了自定义布局到系统布局的添加。
接下来我们看DecorView是如何显示出来的，刚开始学习Android的时候，我们都听说过布局是在Activity的onResume之后才会显示出来的，我们不妨去看下``ActivityThread.handleResumeActivity``中找下答案。
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffewqfxi2qj31kw0gcn2a.jpg)
先去看看``performResumeActivity``中做了什么
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffews5a827j31b40pewm1.jpg)
可以看到调用了Activity的``performResume``方法。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffewtq4dfoj30ym0b0jti.jpg)
**Instrumentation.callActivityOnResume(Activity activity)**
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffewvkbwg6j31ck0m6wk4.jpg)
可以看到这里回调了``Activity.onResume``方法。再回到``ActivityThread.handleResumeActivity``中来，接着往下看。
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffewy1zb8pj318c0r2ain.jpg)
可以看到``WindowManager.addView(decorView, layoutParams)``,``WindowManager``是个interface，我们去具体的实现类``WindowManagerImpl``中去看下。
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffex0nslm8j31fk066mz4.jpg)
**WindowManagerGlobal.addView**
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffex2p99gqj318a0ekn03.jpg)
可以看到实例化了一个ViewRootImpl,并且调用了``ViewRootImpl.setView``,
在``ViewRootImpl.setView``中调用了``requestLayout()``和``view.assignParent(this)``。``view.assignParent``将DecorView的parent设置为了``ViewRootImpl``.``View``的``requestLayout``中调用``getParent().requestLayout()``，所以，布局中任何一个view的``requestLayout()``方法调用都会调到``ViewRootImpl``的``requestLayout``.
接下来看``ViewRootImpl.requestLayout``
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffex9sawz5j30tm09gdhd.jpg)
**ViewRootImpl.scheduleTraversals**
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffexaw8otmj31dm0eegpm.jpg)
Choreographer.postCallback posts a callback to run on the next frame.Choreographer主要是同步VSYNC信号用的，现在我们去看下postCallback中的``mTraversalRunnable``.
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffexej6xnoj3180064q4g.jpg)
继续看``doTraversal``主要调用了``performTraversals``方法。在``performTraversals``方法中依次调用了``performMeasure``,``performLayout``,``performDraw``方法。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffexj2s85zj31fg0ja46p.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffexjurpdfj31dw06ognl.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffexkoh68jj31e00mgtdx.jpg)



