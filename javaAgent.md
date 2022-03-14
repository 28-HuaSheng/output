### javaAgent
* 源自MegaEase分享 
* https://github.com/megaease
* 依赖技术：https://github.com/raphw/byte-buddy
* agent介绍：https://ken-ljq.github.io/2018/JVM%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7Agent%E8%AE%BE%E8%AE%A1%E5%AE%9E%E7%8E%B0-%E4%BA%8C/

### javaAgent 加载
* <img width="749" alt="image" src="https://user-images.githubusercontent.com/46739345/158030545-529fdeef-ea5f-4394-98b1-3fb289e9a9ef.png">

### byteBuddy增强示例
* <img width="499" alt="image" src="https://user-images.githubusercontent.com/46739345/158030761-72bd6a7a-6faf-4248-94ab-1e3f75fad1e7.png">

### 实践问题
#### 1.依赖冲突
* java agent依赖 z 包版本 与 应用程序依赖 z 包版本不同导致冲突
* 问题原因
  1. 默认用户类都是使用 Appclassloader加载,导致异常
* 解决方案
  1. maven-shade-plugin
  2. 自定义classloader（将原本相同的类加载器进行隔离，使用自定义类加载器）
    * 类加载过程(双亲)
      <img width="486" alt="image" src="https://user-images.githubusercontent.com/46739345/158031130-12cd0eff-faff-493d-af94-8eb71f0e9eb9.png">
    * 通过agent1的premain方法内自定义类加载器加载agent2类（附带agent2使用到的类），通过反射执行agent2的 premain方法
      <img width="835" alt="image" src="https://user-images.githubusercontent.com/46739345/158031252-59a70a8a-b871-45f3-8881-9ce1c2e671bd.png">

#### 2.class not found
* 在加载完agent classes和用户classes后，获得增强后的字节码，增强后的字节码通过app class loader加载过程中 发现某些类不可得。
* 增强代码由agent提供，和用户代码一起由app class loader加载。
* 解决方案：

