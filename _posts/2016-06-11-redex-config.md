#Redex简介
Redex是Facebook提供的Android字节码优化工具，可以获得更小的dex，并且通过区分启动、非启动的类，获得启动速度优化。

[Facebook App 优化工具 ReDex 优化的 6 点及未优化的一大方面](http://www.trinea.cn/android/facebook%E5%BC%80%E6%BA%90%E7%9A%84android%E4%BC%98%E5%8C%96%E5%B7%A5%E5%85%B7redex-%E5%87%8F%E5%B0%8F%E5%AE%89%E8%A3%85%E5%8C%85%E5%A4%A7%E5%B0%8F-%E5%90%8C%E6%97%B6%E6%8F%90%E9%AB%98%E8%BF%90/)

因为目前关于Redex的实际使用方面和文档比较少，这里主要说明Redex支持的配置，以及对应配置和优化项的关系。

#Redex的配置
Redex的配置分有两种方式：
1）命令行输入的配置
2）json配置文件

##命令行配置
通过usage，可以获得具体参数，不再做具体描述。
这里指出几个关键的配置：

--proguard-config
可以指定apk所使用的proguard文件，用于转换原始类名与混淆后的类名转换。

-- seeds 
指定apk的seeds文件，作为保持类的依据。

-- config 
用于指定配置文件的位置，后边详述。
配置分为全局配置，针对对应优化项的配置，都可以听过key-value的方式或者key=<json-value>进行配置。

全局配置，可以直接指定对应的key。

针对优化项，例如针对RenameClassesPass这个优化项，就可以使用这种方式：
-SRenameClassesPass.class_rename=/foo/bar/data.txt
-JRenameClassesPass.pre_filter_whitelist=["1", "2"]

##json配置文件
因为各个优化项可能会有自己的特殊配置，这里只列出对全局redex支持的配置。

##启动优化相关的配置
Redex其实是要求提供启动相关的类和方法，使得这些类和方法，被放入同一个dex中，从而提高启动速度。

|配置项名称|说明|取值|
|---|---|---|
|proguard_map|proguard文件的对应路径| /path/proguard.cfg|
|coldstart_classes|指定类启动相关类的文件，一个类一行| /path/classes.list |
|coldstart_methods|指定类启动相关方法的文件，一个方法一行| /path/methods.list |

##优化项相关配置
|配置项名称|说明|取值|
|---|---|---|
|no_optimizations_annotations|在优化中，对应的注解类会被保留|["1","2"]|
|keep_annotations|在优化中，对应的注解类会被保留|["1","2"] |
|keep_packages|在优化中，保留以此包名为开头的类|["1","2"] |
|keep_class_members|支持静态变量名的保留,格式：类名静态变量名| ["ClassNameSFiedlsName","ClassNameSFiedlsName"]|


#Redex配置的源码分析

从main函数作为入口，主要经过以下步骤：
create_passes
parse_args
load_proguard_config_file
load_classes_from_dex
init_seed_classes
run_passes
write_classes_to_dex

从main函数作为入口，会经过create_passes阶段，这一步的优化步骤和其执行顺序是固定的，在配置文件里边的配置，只是配置哪些步骤会被激活，而执行顺序只和代码内置的熟悉怒有关。如下:
```
std::vector<Pass*> create_passes() {
  return std::vector<Pass*>({
    new AnnoClassKillPass(),
    new AnnoKillPass(),
    new BridgePass(),
    new ConstantPropagationPass(),
    new DelInitPass(),
    new DelSuperPass(),
    new FinalInlinePass(),
    new InterDexPass(),
    new LocalDcePass(),
    new PeepholePass(),
    new ReBindRefsPass(),
    new RemoveEmptyClassesPass(),
    new RenameClassesPass(),
    new ShortenSrcStringsPass(),
    new SimpleInlinePass(),
    new SingleImplPass(),
    new StaticSinkPass(),
    new StaticReloPass(),
    new SynthPass(),
    new UnterfacePass(),
  });
}
```

通过parse_args，会得到Argemetns，包括了json格式的配置文件 ，jar路径，proguard\seed 的文件路径，apk输出的目录。
而上一节所提到的“json配置文件 ”，就是通过Argemetns一路向下传递。

load_proguard_config_file，会把从proguard文件中，独到的信息，转换成KeepRule的数组。从而，为后边保留规则做准备。虽然KeepRule是一个基本的proguard文件信息反映，但后边的优化项，并不是100%遵循这个规则进行保留，因为Redex团队认为，proguard规则，往往是过度保护，导致无法得到最好的优化。因此，并不是所有被读取的规则所需要保护的类和方法，会被保留。而且，目前，也只是支持interafece, class，需要指定到类名，不支持通配符。
KeepRule的包含信息如下：

load_classes_from_dex会从dex文件中，把对应的代码，构建成DexClasses，为后续的优化提供输入，这一部分也是以后需要单独讲的。

init_seed_classes，会从seed文件逐行取出类型，和Dex中的内容作对比，如果可以找到需要保存的类型，就会设置到ConfigFiles结构中，以便日后使用seeds信息。

根据输入的配置信息，到底哪些类和方法会被保留呢？哪些不会保留。
目前这部分的逻辑主要在每次优化项调用前，会使用ReachableClasses的init_reachable_classes方法。它会包含两部工作：
1）永久保留的类：主要是根据输入，包括之前的json配置文件、AndroidManifest.xml，KeepRule规则，来做匹配，因此，相当于使用了proguard\seed的规则做保留。
2）根据每次优化项之后，代码的引用情况来做保留标记：根据代码和资源引用的情况来做。

目前看到，参考ReferencedState类，Redex里边区分了三种引用情况：
1）被类型引用，相当于被代码引用，Redex认为这种情况是不能被删除，can_delete函数返回false
2）被字符串引用，相当于有反射和资源引用的情况，认为不能被重命名。can_rename函数返回false
3）被seed引用，seed文件中要求被保留的情况。is_seed返回true

优化项和这三类的引用情况对应关系如下，可以看到其他优化项是有可能忽略引用关系，而无法被保留的情况。

|优化项|    can_delete|    can_rename|    is_seed|
|---|---|---|---|
| InterDex|    *  |   *    |
| RenameClasses |    *|    *    |
| FinalInline |    *  |     
| LocalDce |    *  |     
| RemoveEmpytClasses  |   *    |   
| Deleter |    *   |     
| SimpleInline |    *   |     
| SimpleImplAnalyze |    *   |     
| StaticSink |    *   |     
| StaticRelo   | *   | |     * |
| Synth  |   *    |   


#结论
目前Redex对proguard的支持不是完整支持，而且存在优化项忽略保留规则的情况，而进行更多优化。
因此在配置优化项的时候需要注意，以免因为优化导致需要的选项没被优化下来。

后续文章，会给出一个具体的完整配置例子的实例，以及优化项的详细介绍。
