```
// 方法定义：inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
View view = LayoutInflater.from(context).inflate(R.layout.resource,root,flase);
```

这样我们就可以把一个 XML 文件实例化成一个 View 来供我们使用。LayoutInflater的获取方式不止这一种，实际最终调用的都是 Context.getSystemService 方法，最终拿到的是``PhoneLayoutInflater``.

![](http://ww1.sinaimg.cn/large/006tNbRwly1ffpt3uv84bj31bc0fyn1w.jpg)
![](http://ww4.sinaimg.cn/large/006tNbRwly1ffpt495mdpj31kw0wpdrd.jpg)

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        // 一些赋值 后续会有用到
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        // 先把result赋值为我们传递的root
        View result = root;
        try {
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            // 获取标签名字 比如FrameLayout
            final String name = parser.getName();
            
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }
            // 这里的 TAG_MERGE 为 merge 看到了merge的身影
            if (TAG_MERGE.equals(name)) {
                // 不知道merge怎么用？ 这个异常教你做人。
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                //如果为 merge 调用rInflate方法 后面再具体分析merge的情况
                //注意最后一个参数是false
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                // 这里调用了一个createViewFromTag 从名字来看，就是用来创建View的！
                // 注意，这里的temp其实是我们xml里的top view,具体暂时先不管 先把整个流程看了
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                //接下去处理LayoutParams
                ViewGroup.LayoutParams params = null;
                //如果我们传递进来的root不为null              
                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    // 那么调用 root的generateLayoutParams 来生成LayoutParamas
                    params = root.generateLayoutParams(attrs);
                    //如果attachToRoot为false，那么就把刚生成的params赋值给View
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // 源码的打印日志已经告诉我，这里是加载子View的~~ 后续再讲解
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);
                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }
                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                // root不为null 并且 attachToRoot 则直接把temp添加到root里去
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                // null或false 那么结果就是之前的top view了
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            InflateException ex = new InflateException(e.getMessage());
            ex.initCause(e);
            throw ex;
        } catch (Exception e) {
            InflateException ex = new InflateException(
                    parser.getPositionDescription()
                            + ": " + e.getMessage());
            ex.initCause(e);
            throw ex;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        // 返回结果
        return result;
    }
}
```
LayoutParamas 一般有3种来源：
+ 用户完全自定义 自己 new 出来
+ ViewGroup.generateLayoutParams 方法生成 上面已经提到
+ ViewGroup.generateDefaultLayoutParams 方法生成，在addView的时候 如果 childView 没有的话 LayoutParams 属性的话，会由这个方法生成。

来看一下 addView 里对 paramas 的操作就明白了：

```
public void addView(View child) {
    addView(child, -1);
}
public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        // 没有 params 则调用 generateDefaultLayoutParams 去生成
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}
```
createViewFromTag解析

```
/**
 * 提到了include，除了include，都会被使用
 * Convenience method for calling through to the five-arg createViewFromTag
 * method. This method passes {@code false} for the {@code ignoreThemeAttr}
 * argument and should be used for everything except {@code &gt;include>}
 * tag parsing.
 */
private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
    return createViewFromTag(parent, name, context, attrs, false);//这里传了false进去
}
```

该方法把ignoreThemeAttr属性赋值为了false

```
/**
 * Creates a view from a tag name using the supplied attribute set.
 * @param ignoreThemeAttr {@code true} to ignore the {@code android:theme}
 *                        attribute (if set) for the view being inflated,
 *                        {@code false} otherwise
 */
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }
    // Apply a theme wrapper, if allowed and one is specified.
    // 之前传递过来的为false
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        //如果设置了theme 那么context会被重新实例化为 ContextThemeWrapper
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }
    // 彩蛋后续 分析
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }
    // 这里开始去创建View了
    try {
        View view;
        //如果mFactory2不为null 那么mFactory2先解析
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            // 接着是 mFactory
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }
        // 如果 mFactory2 mFactory都返回null了 那么如果mPrivateFactory不为null，则交给它
        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
        // 如果 那几个factory都返回null 即view还是null 那么继续
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                // 有. 代表不是系统自带的View,比如TextView、me.yifeiyuan.XXXLayout
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    // 系统自带的View
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
        return view;
    } catch (InflateException e) {
        throw e;
    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;
    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;
    }
}
```

createViewFromTag方法 先处理了主题属性，再走入创建View的流程。这里还涉及到了几个Factory，这其实是系统留给我们的Hook入口，我们可以人为的干涉系统创建View，可以添加更多功能，比如夜间模式。

```
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    // 构造方法缓存
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;
    try {
        // Trace
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
        //没有缓存 则去获取constructor 并存入缓存 注意这个缓存是静态的
        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            // 注意 注意 注意 在这里把 View的全称拼全，并loadClass 难怪要传递android.View.进来啊~
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            // 让 Filter 处理这个clazz能否被加载
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    // 如果不允许加载 则failNotAllowed会抛出异常！
                    failNotAllowed(name, prefix, attrs);
                }
            }
            // 反射获取构造方法 并存入缓存
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // 如果有缓存 就走filter流程，并把结果存入缓存（非静态）
            // If we have a filter, apply it to cached constructor
            if (mFilter != null) {
                // Have we seen this name before?
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // New class -- remember whether it is allowed
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);
                    // 获取allowed 并存入 缓存
                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }
        // mConstructorArgs存放了context 跟 args
        Object[] args = mConstructorArgs;
        args[1] = attrs;
        // 终于看到view实例化的地方了！！
        final View view = constructor.newInstance(args);
        // 如果是ViewStub 则设置LayoutInflater给它用 
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;
    } catch (NoSuchMethodException e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;
    } catch (ClassCastException e) {
        // If loaded class is not a View subclass
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Class is not a View "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;
    } catch (ClassNotFoundException e) {
        // If loadClass fails, we should propagate the exception.
        throw e;
    } catch (Exception e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (clazz == null ? "<unknown>" : clazz.getName()));
        ie.initCause(e);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

看到这方法的 clazz = mContext.getClassLoader().loadClass(prefix != null ? (prefix + name) : name).asSubclass(View.class) 步骤会把系统自带的View的路径拼起来，把类加载进来；

然后clazz.getConstructor(mConstructorSignature);获取View的构造方法，最终通过反射constructor.newInstance(args);实例化View。

WebView 怎么办？它的路径可是android.webkit啊~
其实这里涉及到 LayoutInflater 的一个子类com.android.internal.policy.PhoneLayoutInflater，它处理了android.widget.、android.webkit.、android.app.这些路径。

merge 标签分析

```
if (TAG_MERGE.equals(name)) {
    if (root == null || !attachToRoot) {
        throw new InflateException("<merge /> can be used only with a valid "
                + "ViewGroup root and attachToRoot=true");
    }
    // 调用 rInflate 注意最后的参数是 false
    rInflate(parser, root, inflaterContext, attrs, false);
}
```

rInflate 深入解析

```
/**
 * Recursive method used to descend down the xml hierarchy and instantiate
 * views, instantiate their children, and then call onFinishInflate().
 * <p>
 * <strong>Note:</strong> Default visibility so the BridgeInflater can
 * override it.
 * 递归方法 实例化 View 以及它的子 View,并且调用 onFinishInflate
 */
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    final int depth = parser.getDepth();
    int type;
    // while 循环 parser.next  遍历整个 XML 
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
        if (type != XmlPullParser.START_TAG) {
            continue;
        }
        //获取标签名
        final String name = parser.getName();
        //如果是 requestFocus
        if (TAG_REQUEST_FOCUS.equals(name)) {
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {
        	// 处理 tag
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            // 处理 include
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
        	// merge 里不能再有 merge 标签的
            throw new InflateException("<merge /> must be the root element");
        } else {
        	// 如果不是特殊的标签那么走 createViewFromTag
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }
    // 注意这里从传递inflate传递过来的是 false！！
    // 从 rInflateChildren 过来的是 true！！！
    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

从源码来看，rInflate方法先判断特殊的标签名，优先处理：

+ 针对requestFocus标签,调用parseRequestFocus方法
+ 针对tag标签，调用 parseViewTag 方法
+ 针对merge标签则直接抛了异常，因为merge标签不能是子元素

处理完特殊标签之后走到最后一个else 块中，这块代码需要注意：

```
// 如果不是特殊的标签那么走 createViewFromTag 获得 view
// createViewFromTag 方法的流程已经分析过了，不再多说。  
final View view = createViewFromTag(parent, name, context, attrs);
// 注意注意 这边的 parent 是之前inflate传入的 root
final ViewGroup viewGroup = (ViewGroup) parent;
// 生成 paramas
final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
// 调用 rInflateChildren 把 view传递过去 并传了一个 true 过去
rInflateChildren(parser, view, attrs, true);
// 注意
// 注意
// 注意
// 这里直接调用 viewGroup 的 addView 这也是 merge 能减少层级的根本原因
viewGroup.addView(view, params);
```

```
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```
rInflateChildren 是个递归方法，被用来实例化不是 root 的子View，实际也是调用rInflate方法。(所以才是递归了嘛！)
可以看到，在处理merge标签的时候，是将merge标签里解析出来的 View 直接 add 到了传递进来的root中去了，而并不会多加一层 View，从而实现减少层级的效果，这就是merge标签的原理所在了。
最后，由于inflate传递进来的finishInflate为 false，所以不会去调用parent.onFinishInflate();

include 标签

```
private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    int type;
    if (parent instanceof ViewGroup) {
        // Apply a theme wrapper, if requested. This is sort of a weird
        // edge case, since developers think the <include> overwrites
        // values in the AttributeSet of the included View. So, if the
        // included View has a theme attribute, we'll need to ignore it.
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        final boolean hasThemeOverride = themeResId != 0;
        if (hasThemeOverride) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
        // If the layout is pointing to a theme attribute, we have to
        // massage the value to get a resource identifier out of it.
        int layout = attrs.getAttributeResourceValue(null, ATTR_LAYOUT, 0);
        if (layout == 0) {
        	// 没有 layout 属性则抛异常
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            if (value == null || value.length() <= 0) {
                throw new InflateException("You must specify a layout in the"
                        + " include tag: <include layout=\"@layout/layoutID\" />");
            }
            // Attempt to resolve the "?attr/name" string to an identifier.
            layout = context.getResources().getIdentifier(value.substring(1), null, null);
        }
        // The layout might be referencing a theme attribute.
        if (mTempValue == null) {
            mTempValue = new TypedValue();
        }
        if (layout != 0 && context.getTheme().resolveAttribute(layout, mTempValue, true)) {
            layout = mTempValue.resourceId;
        }
        // 之前的代码都是处理 theme layout 属性 不多说。
        if (layout == 0) {
        	// 必须指定有效的layout
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            throw new InflateException("You must specify a valid layout "
                    + "reference. The layout ID " + value + " is not valid.");
        } else {
            final XmlResourceParser childParser = context.getResources().getLayout(layout);
            try {
                final AttributeSet childAttrs = Xml.asAttributeSet(childParser);
                while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty.
                }
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(childParser.getPositionDescription() +
                            ": No start tag found!");
                }
                final String childName = childParser.getName();
                if (TAG_MERGE.equals(childName)) {
                    // The <merge> tag doesn't support android:theme, so
                    // nothing special to do here.
                    rInflate(childParser, parent, context, childAttrs, false);
                } else {
                	// 获取 被 inlcude 的 topview
                    final View view = createViewFromTag(parent, childName,
                            context, childAttrs, hasThemeOverride);
                    final ViewGroup group = (ViewGroup) parent;
                    final TypedArray a = context.obtainStyledAttributes(
                            attrs, R.styleable.Include);
                    final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
                    final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                    a.recycle();
                    // We try to load the layout params set in the <include /> tag.
                    // If the parent can't generate layout params (ex. missing width
                    // or height for the framework ViewGroups, though this is not
                    // necessarily true of all ViewGroups) then we expect it to throw
                    // a runtime exception.
                    // We catch this exception and set localParams accordingly: true
                    // means we successfully loaded layout params from the <include>
                    // tag, false means we need to rely on the included layout params.
                    ViewGroup.LayoutParams params = null;
                    try {// 尝试对 include 标签生成 params
                        params = group.generateLayoutParams(attrs);
                    } catch (RuntimeException e) {
                        // Ignore, just fail over to child attrs.
                    }
                    // 如果失败 则对被 include 的 topview 处理
                    if (params == null) {
                        params = group.generateLayoutParams(childAttrs);
                    }
                    view.setLayoutParams(params);
                    // Inflate all children.  前面已经提到过了
                    rInflateChildren(childParser, view, childAttrs, true);
                    // 处理 id
                    if (id != View.NO_ID) {
                        view.setId(id);
                    }
                    // 处理可见性
                    switch (visibility) {
                        case 0:
                            view.setVisibility(View.VISIBLE);
                            break;
                        case 1:
                            view.setVisibility(View.INVISIBLE);
                            break;
                        case 2:
                            view.setVisibility(View.GONE);
                            break;
                    }
                    // 把 view 添加到 group 中
                    group.addView(view);
                }
            } finally {
                childParser.close();
            }
        }
    } else {
        throw new InflateException("<include /> can only be used inside of a ViewGroup");
    }
    LayoutInflater.consumeChildElements(parser);
}
```

首先判断 parent 是不是个 ViewGroup,如果不是则直接抛异常。

如果是则接下去处理 theme 属性以及 layout 属性，我们知道使用include标签，layout 属性是必须要有的。

其原因就是在源码中如果发现没有指定 layout 属性的话，那么会直接抛出异常。

再接下去的步骤可以看出其实跟上篇inflate方法类似：

+ 通过调用createViewFromTag解析获取被include的 topview
+ 生成 params，这里要注意，include标签可能没有宽高，会导致生成失败，如果失败则接着又对被include的 topview 做操作。所以使用include的时候，不对它设置宽高是没有关系的。
+ 调用rInflateChildren处理子View 之前已经分析过
+ 把 include 标签的 id 以及 visibility属性 设置给 topview（如果有的话）
+ topView 被直接 add 进 group，这样被 include 的 topView 就被加到布局里去了。

```
<fragment
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    class="me.yifeiyuan.MainFragment"
    android:tag="Main"
    android:id="@+id/main"
    />
```

寻找踪迹之 FragmentActivity
![](http://ww1.sinaimg.cn/large/006tNbRwly1ffpunqsz68j30fq0r0wfs.jpg)

其实 Activity 就已经实现了 LayoutInflater.Factory2 接口
![](http://ww3.sinaimg.cn/large/006tNbRwly1ffpuouk6twj31kw0xdk43.jpg)

但是需要注意的是：Activity 并没有调用 LayoutInflater.setFactory 使之生效，所以 Activity 并不支持 fragment 的解析。

```
abstract class BaseFragmentActivityDonut extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        if (Build.VERSION.SDK_INT < 11 && getLayoutInflater().getFactory() == null) {
            // On pre-HC devices we need to manually install ourselves as a Factory.
            // On HC and above, we are automatically installed as a private factory
            getLayoutInflater().setFactory(this);
        }
        super.onCreate(savedInstanceState);
    }
    //重写 onCreateView
    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
    	//　优先调用 dispatchFragmentsOnCreateView
        final View v = dispatchFragmentsOnCreateView(null, name, context, attrs);
        if (v == null) {
            return super.onCreateView(name, context, attrs);
        }
        return v;
    }
    abstract View dispatchFragmentsOnCreateView(View parent, String name,
            Context context, AttributeSet attrs);
}
// Honeycomb
abstract class BaseFragmentActivityHoneycomb extends BaseFragmentActivityDonut {
    @Override
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        final View v = dispatchFragmentsOnCreateView(parent, name, context, attrs);
        if (v == null && Build.VERSION.SDK_INT >= 11) {
            // If we're running on HC or above, let the super have a go
            return super.onCreateView(parent, name, context, attrs);
        }
        return v;
    }
}
// FragmentActivity 并没有重写 onCreateView
public class FragmentActivity extends BaseFragmentActivityHoneycomb{
    @Override
    final View dispatchFragmentsOnCreateView(View parent, String name, Context context,
            AttributeSet attrs) {
        return mFragments.onCreateView(parent, name, context, attrs);
    }
}
```

可以看到，在BaseFragmentActivityDonut中调用了setFactory，并定义了一个dispatchFragmentsOnCreateView抽象方法，并在onCreateView里调用了它，这样就把创建 View 的工作交给了dispatchFragmentsOnCreateView。

接着，在 FragmentActivity 重写dispatchFragmentsOnCreateView，又把它交给了mFragments，咦，又回去了。

去看下``mFragments.onCreateView(parent, name, context, attrs);``

![](http://ww4.sinaimg.cn/large/006tNbRwly1ffpv3dygh2j31kg0fotdj.jpg)

最终会调用``FragmentManagerImpl.onCreateView``

```
    @Override
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        if (!"fragment".equals(name)) {
            return null;
        }

        String fname = attrs.getAttributeValue(null, "class");
        TypedArray a =
                context.obtainStyledAttributes(attrs, com.android.internal.R.styleable.Fragment);
        if (fname == null) {
            fname = a.getString(com.android.internal.R.styleable.Fragment_name);
        }
        int id = a.getResourceId(com.android.internal.R.styleable.Fragment_id, View.NO_ID);
        String tag = a.getString(com.android.internal.R.styleable.Fragment_tag);
        a.recycle();

        int containerId = parent != null ? parent.getId() : 0;
        if (containerId == View.NO_ID && id == View.NO_ID && tag == null) {
            throw new IllegalArgumentException(attrs.getPositionDescription()
                    + ": Must specify unique android:id, android:tag, or have a parent with"
                    + " an id for " + fname);
        }

        // If we restored from a previous state, we may already have
        // instantiated this fragment from the state and should use
        // that instance instead of making a new one.
        Fragment fragment = id != View.NO_ID ? findFragmentById(id) : null;
        if (fragment == null && tag != null) {
            fragment = findFragmentByTag(tag);
        }
        if (fragment == null && containerId != View.NO_ID) {
            fragment = findFragmentById(containerId);
        }

        if (FragmentManagerImpl.DEBUG) Log.v(TAG, "onCreateView: id=0x"
                + Integer.toHexString(id) + " fname=" + fname
                + " existing=" + fragment);
        if (fragment == null) {
            fragment = Fragment.instantiate(context, fname);
            fragment.mFromLayout = true;
            fragment.mFragmentId = id != 0 ? id : containerId;
            fragment.mContainerId = containerId;
            fragment.mTag = tag;
            fragment.mInLayout = true;
            fragment.mFragmentManager = this;
            fragment.mHost = mHost;
            fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
            addFragment(fragment, true);
        } else if (fragment.mInLayout) {
            // A fragment already exists and it is not one we restored from
            // previous state.
            throw new IllegalArgumentException(attrs.getPositionDescription()
                    + ": Duplicate id 0x" + Integer.toHexString(id)
                    + ", tag " + tag + ", or parent id 0x" + Integer.toHexString(containerId)
                    + " with another fragment for " + fname);
        } else {
            // This fragment was retained from a previous instance; get it
            // going now.
            fragment.mInLayout = true;
            fragment.mHost = mHost;
            // If this fragment is newly instantiated (either right now, or
            // from last saved state), then give it the attributes to
            // initialize itself.
            if (!fragment.mRetaining) {
                fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
            }
        }

        // If we haven't finished entering the CREATED state ourselves yet,
        // push the inflated child fragment along.
        if (mCurState < Fragment.CREATED && fragment.mFromLayout) {
            moveToState(fragment, Fragment.CREATED, 0, 0, false);
        } else {
            moveToState(fragment);
        }

        if (fragment.mView == null) {
            throw new IllegalStateException("Fragment " + fname
                    + " did not create a view.");
        }
        if (id != 0) {
            fragment.mView.setId(id);
        }
        if (fragment.mView.getTag() == null) {
            fragment.mView.setTag(tag);
        }
        return fragment.mView;
    }

    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        return null;
    }
```

可以看到，在处理 xml 中的属性后，会先去寻找要加载的 fragment 是否已经加载过了，如果没有则会调用``fragment = Fragment.instantiate(context, fname);``，这个方法也是
反射，这点其实跟 View 的处理是一样的。

接着会去调用``moveToState``方法，而这个方法里，我看了``onCreateView``方法的调用时机。

```
// # FragmentManager
void moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive) {
	//...
	f.mView = f.performCreateView(xxx);
	//....
}
// 真正的调用时机在这里！！
// # Fragment
View performCreateView(LayoutInflater inflater, ViewGroup container,
        Bundle savedInstanceState) {
    if (mChildFragmentManager != null) {
        mChildFragmentManager.noteStateNotSaved();
    }
    // 调用onCreateView 熟悉吧？
    return onCreateView(inflater, container, savedInstanceState);
}
```

可以看到，FragmentManager的moveToState中会去调用Fragment的performCreateView方法，而它里面，调用了onCreateView！！

FragmentActivity通过 setFactory把对fragment标签的处理委托给了 FragmentManageImpl的onCreateView方法。

最终通过反射，实例化指定的 Fragment，并调用了Fragment.performCreateView，最后到我们所熟悉的onCreateView。

另外要说的是，LayoutInflater.Factory的作用其实非常强大，我们可以 Hook 每个 View 的创建于设置，比如 AppCompact库通过AppCompactViewInflater Hook 了大部分 View，给我们提供了向下兼容的功能；

另外它也还可以配合 DayNight 实现夜间模式功能，有兴趣可以去看看AppCompactActivity、AppCompactViewInflater等类.