title: Android - 支持multidex后NoClassDefFoundError的解决办法
tags: Android multidex NoClassDefFoundError
---

## 问题
Android应用支持multiDex以后，在某些5.0以下的机型上，会遇到某些类由于没有被打进MainDex中，而导致的 NoClassDefFoundError 错误。

***更多关于 ClassNotFoundException 和 NoClassDefFoundError 的讨论 [可以到这里查看](http://stackoverflow.com/a/1457879/5571166).***

## 解决方式
### 过时的方式
在使用Android gradle build tools 1.2.3 版本的时候，我们在 build.gradle 的 `defaultConfig` 中加入一个非公开属性 `multiDexKeepFile`
> multiDexKeepFile file("multidex.keep")

这个`multidex.keep`的内容，是要保持类在MainDex中的Class列表，如：

	com/alibaba/mobileim/channel/util/SimpleKVStore.class
	com/alibaba/mobileim/channel/util/SimpleKVStore$SingletonHolder.class
	android/webkit/JniUtil.class
	mtopsdk/xstate/XState.class
	android/app/ANRManagerProxy.class

### 新的方式
但是随着升级 build tools 到 1.3.+ 的版本以后，这个功能就失效了。

#### 分析
查询了代码 1.2.3 和 1.3.1 的 build tools源码后，发现原来在 1.2.3版本 [TaskManager.groovy](https://android.googlesource.com/platform/tools/base/+/gradle_1.2.3/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/TaskManager.groovy) 中 createPostCompilationTasks()方法里读取 `multiDexKeepFile`属性的代码都被干掉了。
> File multiDexKeepFile = config.getMultiDexKeepFile();


1.3.0 及以后的版本中，优化了代码结构，设计了一些[新的用于处理MultiDex的Task类](https://android.googlesource.com/platform/tools/base/+/gradle_1.3.1/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/tasks/multidex/):

	CreateMainDexList.groovy
	CreateManifestKeepList.groovy
	JarMergingTask.groovy
	RetraceMainDexList.groovy

通过查看[CreateMainDexList.groovy](https://android.googlesource.com/platform/tools/base/+/gradle_1.3.1/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/tasks/multidex/CreateMainDexList.groovy) 可以看出这个任务最终
会生成一个存储了所有应该在MainDex中的类列表的文件：`maindexlist.txt`。


生成的文件路径是从 [VariantScope.java ](https://android.googlesource.com/platform/tools/base/+/gradle_1.3.1/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/scope/VariantScope.java) 的`getMainDexListFile()`中获取的:
```java
	@NonNull
    public File getMainDexListFile() {
        return new File(globalScope.getIntermediatesDir(), "multi-dex/" + 
        getVariantConfiguration().getDirName() + "/maindexlist.txt");
    }
```

最终生成的这个 `maindexlist.txt` 文件会被[DexProcessBuilder.java](https://android.googlesource.com/platform/tools/base/+/gradle_1.3.1/build-system/builder/src/main/java/com/android/builder/core/DexProcessBuilder.java)通过`--multi-dex --main-dex-list `参数传递给Build tools中的 `dx.jar`(在Android sdk跟目录下有个`build-tools`目录，你可以在这里找到这个jar)，`dx.jar`会用这个列表文件生成MainDex。

#### 解决思路
关键的来了，我们知道新的build tools是通过 `createMainDexList` 这个Task来生成的`maindexlist.txt`，然后才给 `dx.jar`去读取生成 MainDex，那么我们是不是可以hack一下构建过程，在 `createMainDexList` 执行完以后，我们去修改`maindexlist.txt`文件，以达到手工设置MainDex内容的目的呢？

#### 代码实现

结合gradle的构建过程，原来的`multidex.keep`文件位置和内容不变，最终实现代码如下：
```groovy
afterEvaluate {
    tasks.matching{
        it.name.startsWith('create') && it.name.endsWith('MainDexClassList')
    }.each { tk ->
        tk.doLast {
            keepMainMultiDex(tk.outputFile);
        }
    }
}

/**
 * 控制MainDex中的class列表
 * 将multidex.keep的内容追加到 maindexlist.txt 中
 * @param outputFile
 */
def keepMainMultiDex(File outputFile){
    File keepFile = file("multidex.keep");
    outputFile << '\n'
    outputFile << keepFile.getText('UTF-8')
}

```

打完收工~


