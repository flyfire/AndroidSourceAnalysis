``public abstract class AsyncTask<Params, Progress, Result>``

![](https://ws2.sinaimg.cn/large/006tKfTcly1ffs8r9nseqj31kw10xgwq.jpg) 

![](https://ws3.sinaimg.cn/large/006tKfTcly1ffs8txqqbnj31jo03wdgp.jpg)

AsyncTask在构造函数中初始化了mWorker和mFuture，并在初始化mFuture时将mWorker传入。mWorker是Callable对象，mFuture是FutureTask对象。

启动任务：

![](https://ws3.sinaimg.cn/large/006tKfTcly1ffs8xma0laj31i00zegxq.jpg) 

```
    /**
     * Executes the task with the specified parameters. The task returns
     * itself (this) so that the caller can keep a reference to it.
     * 
     * <p>This method is typically used with {@link #THREAD_POOL_EXECUTOR} to
     * allow multiple tasks to run in parallel on a pool of threads managed by
     * AsyncTask, however you can also use your own {@link Executor} for custom
     * behavior.
     * 
     * <p><em>Warning:</em> Allowing multiple tasks to run in parallel from
     * a thread pool is generally <em>not</em> what one wants, because the order
     * of their operation is not defined.  For example, if these tasks are used
     * to modify any state in common (such as writing a file due to a button click),
     * there are no guarantees on the order of the modifications.
     * Without careful work it is possible in rare cases for the newer version
     * of the data to be over-written by an older one, leading to obscure data
     * loss and stability issues.  Such changes are best
     * executed in serial; to guarantee such work is serialized regardless of
     * platform version you can use this function with {@link #SERIAL_EXECUTOR}.
     *
     * <p>This method must be invoked on the UI thread.
     *
     * @param exec The executor to use.  {@link #THREAD_POOL_EXECUTOR} is available as a
     *              convenient process-wide thread pool for tasks that are loosely coupled.
     * @param params The parameters of the task.
     *
     * @return This instance of AsyncTask.
     *
     * @throws IllegalStateException If {@link #getStatus()} returns either
     *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
     *
     * @see #execute(Object[])
     */
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

在execute方法里调用了``onPreExecute``方法。

![](https://ws2.sinaimg.cn/large/006tKfTcly1ffs92ad7puj314o0roq8d.jpg)
![](https://ws3.sinaimg.cn/large/006tKfTcly1ffs92qfvewj31i80dedjw.jpg)

``sDefaultExecutor``是``SerialExecutor``,在``SerialExecutor``中，对Runnable进行了包装，执行execute时候，将Runnable放入到队列尾部，并且在finally块中放入执行下一个Runnable的scheduleNext方法体。第一次调用execute时，将Runnable放入队列尾部，mActive为空，调用scheduleNext方法，将Runnable取出，放入线程池中执行，执行完成后会继续调用scheduleNext方法。

![](https://ws1.sinaimg.cn/large/006tKfTcly1ffs9csfl2fj31da0x60zo.jpg) 

会执行Callable的call方法，也就是mWorker的call方法。

![](https://ws2.sinaimg.cn/large/006tKfTcly1ffs9f94dx1j31eg0kudla.jpg)  

中间调用doInBackground获得结果，并在finally块中调用了postResult。

![](https://ws1.sinaimg.cn/large/006tKfTcly1ffs9jkx9mzj319208a0v2.jpg)

将result封装成AsyncTaskResult作为Message的obj，然后发送给Handler处理。 

![](https://ws2.sinaimg.cn/large/006tKfTcly1ffs9mc7jvrj316k0lwn2i.jpg)
![](https://ws1.sinaimg.cn/large/006tKfTcly1ffs9ni33eaj30s609i402.jpg)

根据取消情况调用``onCancelled``或者``onPostExecute``

![](https://ws1.sinaimg.cn/large/006tKfTcly1ffs9p696puj31di0n2agv.jpg)

调用``publishProgress``时候，会发送消息，回调``onProgressUpdate``

在``mWorker``的call方法中，调用doInBackground时，如果发生了异常，会``mCancelled.set(true)``，后续的``isCancelled``判断就是根据``mCancelled``的。

