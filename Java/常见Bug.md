# 常见Bug

## JVM

**Bug1:**

https://blog.csdn.net/sugar_cookie/article/details/100575726

https://www.jianshu.com/p/4e186fc7d045

https://www.jianshu.com/p/92931e6466b3

mac下是jdk版本不一致的问题，而且需要新的jdk版本

```
Attaching to process ID 4531, please wait...
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
```

**解决：**

- 百度keyword => `Can't attach symbolicator to the process`、`mac jmap`

- `which java => java path`，修改 `~` 目录下的 `.zshrc` => 修改环境变量，更改默认jdk版本 => JDK11

```properties
# jdk1.8
# JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_212.jdk/Contents/Home
# PATH=$JAVA_HOME/bin:$PATH:.
# export JAVA_HOME
# export PATH

# jdk 11
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_212.jdk/Contents/Home
PATH=$JAVA_HOME/bin:$PATH:.
export JAVA_HOME
export PATH
```

- JDK11，jmap不可以用了
  - JDK1.8之前 =>  `jmap -heap 850` 
  - JDK1.8之后 => `jhsdb jmap --heap --pid 850`



```xml
.idea => compiler.xml
<!--Error:java: 无效的标记: --add-exports-->
<module name="jvm_demos" options="--add-exports java.base/jdk.internal.org.objectweb.asm=ALL-UNNAMED" />
```

## Springboot

Spring boot 报错  无法找到匹配的bean 所以无法注入。 可是我项目中明明已经增加了 注解，并且明确指定了需要扫描的包。

最后才发现是因为  spring boot 只扫描 启动类 Application  的子目录 ，而我的 启动类 放在了子目录，报错无法找到的 bean 放到了父目录，所以没有被扫描并且初始化。

最后的解决办法是把 Application 启动类挪到 父目录 解决问题

## 前端

js/jq如何获取后台request等对象封装的数据：https://blog.csdn.net/qq_37811638/article/details/80063335

jq获取thymeleaf：https://blog.csdn.net/qq_39940674/article/details/83446537