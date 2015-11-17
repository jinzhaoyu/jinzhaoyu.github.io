title: Android - AppCompat-v7包踩坑记：活动菜单
date: 2015-11-17 20:25:23
tags: Android AppCompat 菜单 
---
#坑
最近为了使用Android 5.0里的一些UI特性，又懒得写一堆不同系统版本的styles文件。就引入了[AppCompat-v7 Support Library](https://developer.android.com/intl/zh-cn/tools/support-library/features.html) ，使用了 [AppCompatActivity](https://developer.android.com/reference/android/support/v7/app/AppCompatActivity.html) ,可是在测试期间就发现了兼容性问题: 小米手机 和 三星S6 无法打开活动菜单(OptionsMenu)。

多扯一点，对于OptionsMenu，[官方文档 ](http://developer.android.com/intl/zh-cn/guide/topics/ui/menus.html) 中说：

> Beginning with Android 3.0, the Menu button is deprecated (some devices don't have one), so you should migrate toward using the action bar to provide access to actions and other options.

千牛Android没有使用 ActionBar 或 ToolBar，因此只能弹出老式的活动菜单。不过活动菜单中的功能在千牛的界面中都是有入口的，活动菜单只是作为一个便捷的入口而提供的。

上面说到了小米手机和三星S6的问题，他们都有一个共同的特点，就是没有Menu键。Menu键都是通过长按某个按键模拟的：小米的菜单按键在新的MIUI中，单击是显示最近任务，长按才是菜单键。而三星S6是长按返回键模拟菜单键。

#分析
为了兼容各个版本的系统，AppCompatActivity中实际是使用了一个 [AppCompatDelegate](http://developer.android.com/reference/android/support/v7/app/AppCompatDelegate.html) 来处理Activity的各个生命周期及事件回调。这个抽象类目前有这几个包访问级别的子类: `AppCompatDelegateImplV7` 、 `AppCompatDelegateImplV11` 、 `AppCompatDelegateImplV14`，他们负责对具体的系统版本做具体的处理，他们之间也存在继承关系。

##小米
通过追踪`dispatchKeyEvent()` 方法，一路下来，追到了 `AppCompatDelegateImplV7.onKeyDownPanel()` 方法，其代码如下：

```
	private boolean onKeyDownPanel(int featureId, KeyEvent event) {
        if (event.getRepeatCount() == 0) {
            PanelFeatureState st = getPanelState(featureId, true);
            if (!st.isOpen) {
                return preparePanel(st, event);
            }
        }

        return false;
    }
```
这里的`if (event.getRepeatCount() == 0) ` 决定了是否显示菜单的Panel。

某个键被长按时，这个键的 getRepeatCount 是会从0开始往上递增的，由于MIUI是在长按设备上的菜单键时才会把菜单键的点击事件交给App，单击菜单键就被系统拦截，显示最近任务。

当Activity收到Key code 为 82 的菜单键事件时，这个`KeyEvent.getRepeatCount()`取到的是已经不是0了，因为长按会把它累加成一个大于0的数字。

正是由于`KeyEvent.getRepeatCount()`大于0，才因为`AppCompatDelegateImplV7.onKeyDownPanel()`方法里的判断，菜单弹出事件而被过滤掉。

知道原因了，那我们就想办法把 `dispatchKeyEvent()` 获取到的KeyEvent中的`getRepeatCount()`值改成 0 就好了。


##三星S6
三星S6是通过长按返回键（Key code = 4）来模拟的菜单键（Key code = 82），在这款手机上的表现是，显示了菜单但是马上又被关闭了。
通过增加日志：


| event.getAction(): 0  |  event.getKeyCode(): 4  |  event.getRepeatCount(): 0|
|---|---|---|
| ...
| event.getAction(): 0  |  event.getKeyCode(): 4  |  event.getRepeatCount(): 10|
| event.getAction(): 0  |  event.getKeyCode(): 82 |  event.getRepeatCount(): 0|
| event.getAction(): 1  |  event.getKeyCode(): 82 |  event.getRepeatCount(): 0|
| event.getAction(): 0  |  event.getKeyCode(): 4  |  event.getRepeatCount(): 0|

*eventAction = 1 表示 KeyUp ， 0 表示 KeyDown*

可以看出在长按返回键的过程中会发出模拟的菜单键的KeyDown 和 KeyUp事件，但是返回键的KeyUp在菜单键的事件完成后才执行。这就要了命了，我菜单刚显示出来，你一个返回键的KeyUp事件把我的菜单就又给关闭了。


#填坑代码
综上分析，解决办法如下：
## AppCompatActivity 的子类
在 AppCompatActivity 子类的 `onCreate()` 方法中指定自定义的 WindowCallback ,代理掉 `dispatchKeyEvent()` ，利用装饰者模式，处理键盘事件，又不影响AppCompatActivity中原来的逻辑：

```
if(getWindow().getCallback() != null) {
	getWindow().setCallback(new AppCompatWindowCallbackWrapper(getWindow().getCallback()));
}
```        

## 实现 WindowCallbackWrapper 
以下代码中有注释，就不多说了

```
package com.taobao.qianniu.common.widget;

import android.support.v7.internal.view.WindowCallbackWrapper;
import android.view.KeyEvent;
import android.view.Window;

import com.taobao.qianniu.common.utils.PhoneInfo;
import com.taobao.qianniu.component.utils.LogUtil;
import com.taobao.qianniu.controller.common.debugmode.DebugController;
import com.taobao.qianniu.controller.common.debugmode.DebugKey;

/**
 * 为了解决Support V7包的 AppCompatActivity 中对菜单事件的兼容问题,
 * 如小米手机和三星S6 通过长按某个按键模拟菜单键的问题
 *
 * @author jinzhaoyu
 */
public class AppCompatWindowCallbackWrapper extends WindowCallbackWrapper {
	private static final String sTAG = "AppCompatWindowCallbackWrapper";

    public AppCompatWindowCallbackWrapper(Window.Callback wrapped) {
        super(wrapped);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        KeyEvent newEvent = event;
        if(DebugController.isEnable(DebugKey.LOG_DEBUG)) {
            LogUtil.d(sTAG, "KeyEvent.getAction:" + event.getAction() + "    getKeyCode:" + event.getKeyCode() + "    getRepeatCount:" + event.getRepeatCount());
        }
        if (PhoneInfo.isXiaoMiMobile()) {
            //解决小米手机，长按菜单键才能调出菜单的问题
            if (event.getAction() == KeyEvent.ACTION_DOWN
                    && event.getKeyCode() == KeyEvent.KEYCODE_MENU
                    && event.getRepeatCount() > 0) {
                //需要将event.mRepeatCount设置为0，才能绕过 AppCompatDelegateImplV7#onKeyDownPanel
                //方法中对repeatCount的条件判断，进而弹出菜单。
                newEvent = KeyEvent.changeTimeRepeat(event, event.getEventTime(), 0);
            }
        } else if (PhoneInfo.isSamsungMobile()) {
            //解决三星S6，长按返回键调出菜单的问题
            if (checkSamsungKeyEvent(event)){
                return true;
            }
        }
        return super.dispatchKeyEvent(newEvent);
    }


    private boolean isSamsungBackLongPressed = false;
    private boolean isSamsungMockMenuKey = false;
    /**
     * 兼容三星手机用长按返回模拟菜单键的问题
     * @param keyEvent
     * @return
     */
    private boolean checkSamsungKeyEvent(KeyEvent keyEvent){
        switch (keyEvent.getAction()){
            case KeyEvent.ACTION_DOWN:
                //三星S6 通过长按返回键模拟菜单键
                if(keyEvent.getKeyCode() == KeyEvent.KEYCODE_BACK && keyEvent.getRepeatCount() > 0){
                    isSamsungBackLongPressed = true;
                }
                //如果在长按返回没有结束时，收到了菜单键的KeyDown，那么就是Mock的菜单
                if(isSamsungBackLongPressed && keyEvent.getKeyCode() == KeyEvent.KEYCODE_MENU){
                    isSamsungMockMenuKey = true;
                }
                break;
            case KeyEvent.ACTION_UP:
                if(keyEvent.getKeyCode() == KeyEvent.KEYCODE_BACK){
                    isSamsungBackLongPressed = false;
                    if(isSamsungMockMenuKey){
                        isSamsungMockMenuKey = false;
                        return true;
                    }
                }
                break;

        }
        return false;
    }
}
```

可能还有其他机型有类似的问题，也只能碰到一例解决一例了。或者让这逆潮流的功能慢慢消失也不一定是件坏事。