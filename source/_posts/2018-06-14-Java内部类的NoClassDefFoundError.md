---
title: Java同名内部类的 NoClassDefFoundError
date: 2018-06-14 12:50:51
tags:
- Java
description: Java 类的命名问题
---

Windows 不区分文件名的大小写，因此你无法在同一个包中创建两个除大小写外类名完全相同的 Java 文件，但是内部类可以。

```java
public class Fuck {
    public static class FuckYou { }
    public static class Fuckyou { }

    public static void main(String[] args) {
        new Fuck.FuckYou();
    }
}
```

而且这段代码可以通过编译，只有在运行时才会抛出异常:

```
Exception in thread "main" java.lang.NoClassDefFoundError: Fuck$FuckYou (wrong name: Fuck$Fuckyou)
```

当然，正常人不会去写这样的代码，但是用一些生成工具（例如antlr）生成代码时，就有可能出现内部类类名撞车的风险。这个时候，你需要仔细读异常信息：一个类是 `Fuck$FuckYou` 而另一个是 `Fuck$Fuckyou` 