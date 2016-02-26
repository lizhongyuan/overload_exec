####1. 问题描述
&emsp;&emsp;使用qtwebview封装了一个简单浏览器，调用QTest接口模拟鼠标动作。当模拟动作执行的过程中，出现网络中断，有小概率造成程序阻塞。

####2. 调试
>gdb调试


&emsp;&emsp;此处m_webview执行了load函数后，如果在154-163之间的窗口期网络正常，则调用了163行的this->m_eventloop.exec()，此时如果网络异常则有可能导致程序在163行不再向下执行。程序阻塞。

&emsp;&emsp;down, 进入到qt库函数qeventloop.cpp中的 int QEventLoop::exec(ProcessEventsFlags flags)函数，发现是在220行while循环体，processEvents的参数为flags|WaitForMoreEvents|EventLoopExec。
>gdb调试，进入qt源码部分

&emsp;&emsp;processEvents函数：
```
bool QEventLoop::processEvents(ProcessEventsFlags flags)
{
    Q_D(QEventLoop);
    if (!d->threadData->eventDispatcher)
        return false;
    if (flags & DeferredDeletion)
        QCoreApplication::sendPostedEvents(0, QEvent::DeferredDelete);
    return d->threadData->eventDispatcher->processEvents(flags);
}
```
&emsp;&emsp;根据eventDispatcher，查看QEventDispatcherGlib代码(qeventdispatcher_glib.cpp文件)
```

bool QEventDispatcherGlib::processEvents(QEventLoop::ProcessEventsFlags flags)

...

425     bool result = g_main_context_iteration(d->mainContext, canWait);
426     while (!result && canWait)
427         result = g_main_context_iteration(d->mainContext, canWait);


```
&emsp;&emsp;第427行，如果result永远为false，则跳不出循环体。问题就出在这里。
>相关资料：https://developer.gnome.org/glib/2.30/glib-The-Main-Event-Loop.html
&emsp;&emsp;Single iterations of a GMainContext can be run with g_main_context_iteration(). In some cases, more detailed control of exactly how the details of the main loop work is desired, for instance, when integrating the GMainLoop with an external main loop. In such cases, you can call the component functions of g_main_context_iteration() directly. These functions are g_main_context_prepare(),g_main_context_query(), g_main_context_check() and g_main_context_dispatch().

####3. 解决这个问题
&emsp;&emsp;为QEventLoop::exec()添加一个重载函数exec(int waitTime);对应的processEvents也添加waitTime参数。
425至427部分代码修改为：
```
434     bool result = false;
435 
436     while (!result)
437     {
438         time_t cur_time ;
439         time(&cur_time);
440         double diff_time = difftime(b_time, cur_time);
441         if (int(diff_time) > waitTime)
442         {
443             emit awake();
444             result = false;
445             qDebug()<<"TimeOut in glib_processEvents.";
446             break;
447         }
448     
449         result = g_main_context_iteration(d->mainContext, false);
450         if(result)
451             qDebug()<<"g_main_context_iteration return true.";
452     }
```
&emsp;&emsp;这样能大大降低信号丢失（阻塞）的概率（这种修改方式并不能完全解决阻塞，只是极大地降低了发生概率）。


