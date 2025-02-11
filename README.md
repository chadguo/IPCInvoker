# IPCInvoker

[![license](http://img.shields.io/badge/license-Apache2.0-brightgreen.svg?style=flat)](https://github.com/AlbieLiang/IPCInvoker/blob/master/LICENSE)
[![Release Version](https://img.shields.io/badge/release-1.2.6-red.svg)](https://github.com/AlbieLiang/IPCInvoker/releases)
[![wiki](https://img.shields.io/badge/wiki-1.2.6-red.svg)](https://github.com/AlbieLiang/IPCInvoker/wiki) 
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/AlbieLiang/IPCInvoker/pulls)



在Android开发过程中，经常需要写一些跨进程的逻辑，一般情况下我们都是通过AIDL接口来调用的，写AIDL接口并不是一件容易的事情，需要写Service，定义特定的业务接口，调用时需要绑定Service等。

IPCInvoker就是一个用来简化跨进程调用的组件，IPCInvoker底层也是通过AIDL实现的，只是把接口分装得更加容易使用。


## 引入组件库

IPCInvoker组件库已经提交到jcenter上了，可以直接dependencies中配置引用

```gradle
dependencies {
    compile 'cc.suitalk.android:ipc-invoker:<last-version>'
}
```

## 在项目中使用

### 为每个需要支持IPCInvoker的进程创建一个Service


这里以PushProcessIPCService为示例，代码如下：


```java

public class PushProcessIPCService extends BaseIPCService {

    public static final String PROCESS_NAME = "cc.suitalk.ipcinvoker.sample:push";

    @Override
    public String getProcessName() {
        return PROCESS_NAME;
    }
}

```
在manifest.xml中配置service

```xml
<service android:name="cc.suitalk.ipcinvoker.sample.service.PushProcessIPCService" android:process=":push"/>
```

### 在项目的Application中setup IPCInvoker

这里需要在你的项目所有需要支持跨进程调用的进程中调用`IPCInvoker.setup(Application, IPCInvokerInitDelegate)`方法，并在传入的IPCInvokerInitDelegate接口实现中将该进程需要支持访问的其它进程相应的Service的class添加到IPCInvoker当中，示例如下：

```java
public class IPCInvokerApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        // Initialize IPCInvoker
        IPCInvoker.setup(this, new DefaultInitDelegate() {
            
            @Override
            public void onAttachServiceInfo(IPCInvokerInitializer initializer) {
                initializer.addIPCService(PushProcessIPCService.PROCESS_NAME, PushProcessIPCService.class);
            }
        });
    }
}
```
### 在项目代码中通过IPCInvoker调用跨进程逻辑

#### 通过IPCInvoker同步调用跨进程逻辑

```java
public class InvokeSyncSample {

    private static final String TAG = "InvokeSyncSample";

    public static void invokeSync() {
        IPCInteger result = IPCInvoker.invokeSync(
                PushProcessIPCService.PROCESS_NAME, new IPCString("Albie"), HashCode.class);
        Log.i(TAG, "invoke result : %s", result);
    }

    private static class HashCode implements IPCSyncInvokeTask<IPCString, IPCInteger> {

        @Override
        public IPCInteger invoke(IPCString data) {
            return new IPCInteger(data.hashCode());
        }
    }
}
```


#### 通过IPCInvoker异步调用跨进程逻辑

```java
public class InvokeAsyncSample {

    private static final String TAG = "InvokeAsyncSample";

    public static void invokeAsync() {
        IPCInvoker.invokeAsync(PushProcessIPCService.PROCESS_NAME,
                new IPCString("Albie"), HashCode.class, new IPCInvokeCallback<IPCInteger>() {
            @Override
            public void onCallback(IPCInteger data) {
                Log.i(TAG, "onCallback : hascode : %d", data.value);
            }
        });
    }

    private static class HashCode implements IPCAsyncInvokeTask<IPCString, IPCInteger> {

        @Override
        public void invoke(IPCString data, IPCInvokeCallback<IPCInteger> callback) {
            callback.onCallback(new IPCInteger(data.hashCode()));
        }
    }
}
```

上述示例中IPCString和IPCInteger是IPCInvoker里面提供的Parcelable的包装类，IPCInvoker支持的跨进程调用的数据必须是可序列化的Parcelable。

IPCInvoker支持自定义实现的Parcelable类作为跨进程调用的数据结构，同时也支持非Parcelable的扩展类型数据，详细请参考[XIPCInvoker扩展系列接口](https://github.com/AlbieLiang/IPCInvoker/wiki/XIPCInvoker%E6%89%A9%E5%B1%95%E7%B3%BB%E5%88%97%E6%8E%A5%E5%8F%A3)

#### 通过IPCTask实现跨进程调用

IPCTask提供了相对IPCInvoker更为丰富的接口，支持设置连接超时，连接回调，异常回调和确实结果等。与此同时IPCTask支持基本数据类型，并提供了不同于IPCInvoker链式调用方式。

异步调用
```java
public class IPCTaskTestCase {
    
    private static final String TAG = "IPCTaskTestCase";
    
    public static void invokeAsync() {
        IPCTask.create("cc.suitalk.ipcinvoker.sample:push")
                .timeout(10)
                .async(AsyncInvokeTask.class)
                .data("test invokeAsync")
                .defaultResult(false)
                .callback(true, new IPCInvokeCallback<Boolean>() {
                    @Override
                    public void onCallback(Boolean data) {
                        /// callback on UI Thread
                        Log.i(TAG, "invokeAsync result : %s", data);
                    }
                }).invoke();
    }

    private static class AsyncInvokeTask implements IPCAsyncInvokeTask<String, Boolean> {

        @Override
        public void invoke(String data, IPCInvokeCallback<Boolean> callback) {
            callback.onCallback(true);
        }
    }
}
```

同步调用
```java
public class IPCTaskTestCase {

    private static final String TAG = "IPCTaskTestCase";

    public static void invokeSync() {
        Map<String, Object> map = new HashMap<>();
        map.put("key", "value");
        Boolean result = IPCTask.create("cc.suitalk.ipcinvoker.sample:push")
                .timeout(20)
                .sync(SyncInvokeTask.class)
                .data(map)
                .defaultResult(false)
                .invoke();
        Log.i(TAG, "invokeSync result : %s", result);
    }

    private static class SyncInvokeTask implements IPCSyncInvokeTask<Map<String, Object>, Boolean> {

        @Override
        public Boolean invoke(Map<String, Object> data) {
            return true;
        }
    }
}
```

__此外，IPCInvoker还支持跨进程事件监听和分发等丰富的功能，详细使用说明请参考[wiki](https://github.com/AlbieLiang/IPCInvoker/wiki) ，更多使用示例请移步[Sample工程](https://github.com/AlbieLiang/IPCInvoker/tree/master/ipc-invoker-sample)__

## License

```
   Copyright 2017 Albie Liang

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
```
