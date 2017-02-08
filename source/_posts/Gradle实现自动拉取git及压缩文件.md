---
title: Gradle实现自动拉取git及压缩文件
date: 2017-2-1 15:00
categories: Gradle
tags: Gradle
---
### 问题
在实现 hybird 相关开发时，h5文件需要不断重新拉取，并解压文件，拷贝至项目的相关目录下。手工完成比较繁琐且耗时。自己查阅 gradle 相关文档，将其过程实现脚本自动化。
可实现：自动拉取相关 git 服务器上最新文件，并压缩至 src/main/assets/hybird目录下，打包时就会自动带上最新的h5文件了。

### 解决
上代码：

``` gradle
afterEvaluate {
    tasks.matching {
        it.name.startsWith('process') && (it.name.endsWith('ReleaseJavaRes') || it.name.endsWith
                ('DebugJavaRes'))
    }.each { tk ->
        tk.dependsOn(deletehybird)
    }
}

// clone hybird 文件
task cloneHybird(type: Exec){
    delete file("src/main/assets/hybird")

    def osName = System.getProperty("os.name")
    if (osName.contains("Windows")) {
        commandLine 'cmd', '/c', 'git clone  https://github.com/fenglincanyi/…… .git src/main/assets/hybird'
    } else if (osName.contains("Mac OS")) {
        commandLine 'git', 'clone', ' https://github.com/fenglincanyi/…… .git', 'src/main/assets/hybird'
    } else if (osName.contains("LINUX")){
        commandLine 'git', 'clone', ' https://github.com/fenglincanyi/…… .git', 'src/main/assets/hybird'
    }
    println("============== task cloneHybird ==============")
}

// 压缩 hybird文件
task zipHybird(type: Zip) {
    dependsOn cloneHybird

    if (file('src/main/assets/hybird.zip').lastModified() >= file('src/main/assets/hybird').lastModified()) {// 保证zip包最新
        delete file("src/main/assets/hybird.zip")
    }

    archiveName = 'hybird.zip'
    destinationDir = file('src/main/assets')
    from 'src/main/assets/hybird/0.1'
    println("============== task hybirdZip ==============")
}

// 删除 hybird 目录及文件，只留 hybird.zip
task deletehybird(type: Delete) {
    dependsOn zipHybird

    delete "src/main/assets/hybird"
    println("============== deletehybird ==============")
}
```

<br>
此处，单独运行 task，演示效果：
![](http://7xr1vo.com1.z0.glb.clouddn.com/5FAACC49-BCE0-44ED-A60C-30378795BF74.png)

build 结果：
![](http://7xr1vo.com1.z0.glb.clouddn.com/22.png)

在 assets 目录下生成：
![](http://7xr1vo.com1.z0.glb.clouddn.com/33.png)

### 遇到的问题
Mac系统执行 commandLine 去 clone 时，每次都报错：

``` java
Caused by: org.gradle.process.internal.ExecException: A problem occurred starting process 'command 'git clone https://github.com/fenglincanyi/…… .git''
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at org.gradle.process.internal.DefaultExecHandle.setEndStateInfo(DefaultExecHandle.java:197)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at org.gradle.process.internal.DefaultExecHandle.failed(DefaultExecHandle.java:327)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at org.gradle.process.internal.ExecHandleRunner.run(ExecHandleRunner.java:86)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      ... 2 more
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter] Caused by: net.rubygrapefruit.platform.NativeException: Could not start 'git clone https://github.com/fenglincanyi/…… .git'
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at net.rubygrapefruit.platform.internal.DefaultProcessLauncher.start(DefaultProcessLauncher.java:27)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at net.rubygrapefruit.platform.internal.WrapperProcessLauncher.start(WrapperProcessLauncher.java:36)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at org.gradle.process.internal.ExecHandleRunner.run(ExecHandleRunner.java:68)
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      ... 2 more
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter] Caused by: java.io.IOException: Cannot run program "git clone https://github.com/fenglincanyi/…… .git" (in directory "/Users/geng/AndroidStudioProjects/GradleTest/app"): error=2, No such file or directory
11:38:32.043 [ERROR] [org.gradle.BuildExceptionReporter]      at net.rubygrapefruit.platform.internal.DefaultProcessLauncher.start(DefaultProcessLauncher.java:25)
11:38:32.044 [ERROR] [org.gradle.BuildExceptionReporter]      ... 4 more
11:38:32.044 [ERROR] [org.gradle.BuildExceptionReporter] Caused by: java.io.IOException: error=2, No such file or directory
11:38:32.044 [ERROR] [org.gradle.BuildExceptionReporter]      ... 5 more
11:38:32.044 [ERROR] [org.gradle.BuildExceptionReporter]
```
最后Google半天，在 Mac 或 Linux 系统下，需要将命令中的字符串 逐个分割：

``` gradle
commandLine 'git', 'clone', ' https://github.com/fenglincanyi/…… .git', 'src/main/assets/hybird'
```
Gradle 官网的文档，说明的并不是很详细，这里要吐槽下。。。

### 总结
* Gradle 基于Groovy实现，内置了许多好用的API，如：copy，zip等等，这些都可以将传统开发中的手动执行实现自动化
* Gradle 的确方便了开发者，使用脚本来实现繁琐和重复的工作，也将开发过程中配置工作变得更加灵活

<br>
参考：
https://www.jeeboot.com/archives/1563.html
http://stackoverflow.com/questions/15776431/in-gradle-tasks-of-type-exec-why-do-commandline-and-executable-behave-different
https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Exec.html#org.gradle.api.tasks.Exec:commandLine(java.lang.Object%5B%5D
https://segmentfault.com/q/1010000004503896