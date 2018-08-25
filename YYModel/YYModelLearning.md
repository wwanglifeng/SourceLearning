# **前言**
 [YYModel](https://github.com/ibireme/YYModel)是ibireme写的用于模型转换的框架，可从下面几个方面去分析YYModel。
  - 文件结构：有哪些文件，每个文件大致的功能
  - 类结构及关联：有哪些类，类之间关联
  - 数据流向：debug，看数据是如何转换及看各函数之间的调用关系
  - 对重点模块进行剖析 
  - 整理源代码中自己理解不深的点（如语法、关键字的运用）

# **文件结构**
文件可分3部分，总共3000左右行代码
- YYModel.h(.m)  
引入NSObject+YYModel.h 和 YYClassInfo.h，如果要使用YYModel，在对应文件中#import "YYModel.h"。
- YYClassInfo.h(.m)  
类(YYClassInfo)、属性(YYClassPropertyInfo)、函数(YYClassMethodInfo)、实例变量(YYClassIvarInfo)相关的类及赋值操作。
- NSObject+YYModel.h(.m)  
定义了_YYModelPropertMeta、_YYModelMeta类，json数据转Model的实现均在该文件中。

# **类结构及关联**

把主要的类及其关联画了下
![图1](https://github.com/wwanglifeng/SourceLearning/blob/master/images/YYModelClassImage.png)
对照上图，再对各个类做一些简单的介绍
## YYClassIvarInfo
包含Ivar相关的一些信息，如name、offset、typeEncoding，通过initWithIvar函数创建。
```
- (instancetype)initWithIvar:(Ivar)ivar;
```
## YYClassMethodInfo
包含method相关的一些信息，如SEL、IMP、name、typeEncoding、returnTypeEncoding、argumentTypeEncoding，通过initWithMethod函数创建。

```
- (instancetype)initWithMethod:(Method)method;
```
## YYClassPropertyInfo
包含property相关的一些信息，如name、getter、setter、typeEncoding等，通过initWithProperty函数创建。
```
- (instancetype)initWithProperty:(objc_property_t)property;
```
## YYClassInfo
包含class相关的一些信息，如superCls、metaCls、name及上述的3个类。通过initWithClass函数创建。
```
+ (nullable instancetype)classInfoWithClass:(Class)cls;
```
此函数做了2部分工作，1）判断是否有缓存，有的话就取缓存数据 2）没有缓存数据的话通过以下函数进行赋值然后缓存。
```
- (instancetype)initWithClass:(Class)cls;
```
## _YYModelPropertyMeta
类结构中主要包含YYClassPropertyInfo对象、name、mappedToKey等一些相关的信息，用来描述当前实例的属性信息。
通过metaWithClassInfo函数创建
```
+ (instancetype)metaWithClassInfo:(YYClassInfo *)classInfo propertyInfo:(YYClassPropertyInfo *)propertyInfo generic:(Class)generic;
```
## _YYModelMeta
包含了当前类的信息和不同的匹配情况。通过metaWithClass函数创建。metaWithClass中会判断是否有缓存，如果无，就通过调用initWithClass来创建_YYModelMeta,同时讲信息缓存起来，供下次使用
```
+ (instancetype)metaWithClass:(Class)cls;
- (instancetype)initWithClass:(Class)cls;
```


## **数据流向**
看完类的大致结构，再来看下从调用YYModel提供给我们的接口开始，函数之间是如何跳转的，传入的参数又经过了那几道转换。
## yy_modelWithJSON
   该接口接受NSDictionary, NSString,NSData三种类型的json串，返回一个新的是实例对象。实现主要包含以下两部分，重点在第二个函数实现。
```
[self _yy_dictionaryWithJSON:json]  //转成NSDictionary类型数据
[self yy_modelWithDictionary:dic]  //传入NSDictionary数据，返回实例对象
```

来看下yy_modelWithDictionary是如何将dic数据转换成对象的。核心的函数也是两个。

````
//根据调用的类，对YYClassIvarInfo、YYClassMethodInfo、YYClassPropertyInfo、YYClassInfo、_YYModelPropertMeta、_YYModelMeta进行赋值
[_YYModelMeta metaWithClass:cls] 
//对类的属性进行赋值操作
[one yy_modelSetWithDictionary:dictionary]   
````

通过这3个函数就实现了json串到对象的转换,附上一张时序图，看的更直观。
![图2](https://github.com/wwanglifeng/SourceLearning/blob/master/images/YYModel%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

## yy_modelToJSONObject
该函数的流程相对简单点，从_YYModelMeta获取相关的数据赋值给NSDictionary对象。不再做扩展。


# **重点功能剖析**
## 键值的匹配
如果不用第三方框架，自己写代码来做转换，最简单方式如下
```
//省略部分代码
 User *user = [[User alloc] init];
 user.name = json[@"name"];
```
使用YYModel，免去了手动转换的处理。同时YYModel也替我们处理了json串中的数据格式与我们定义的格式不一致的情况，避免了异常的出现。

YYModel还提供了几个protocol，支持匹配特定的JSON格式。
1.指定对于需要匹配的key，或者匹配多个key。
```
+ (nullable NSDictionary<NSString *, id> *)modelCustomPropertyMapper;
```
2.如某个属性中是某个类的数组，这种情况下就需要实现以下protocol。
```
+ (nullable NSDictionary<NSString *, id> *)modelContainerPropertyGenericClass;
```
3.如需根据JSON数据中的返回值，去匹配不同的model就需实现以下protocol。
```
+ (nullable Class)modelCustomClassForDictionary:(NSDictionary *)dictionary;
```
4.YYModel还提供了白名单和黑名单，用户可以在对一些特定的key进行过滤，不参与model转换
```
+ (nullable NSArray<NSString *> *)modelPropertyBlacklist;
+ (nullable NSArray<NSString *> *)modelPropertyWhitelist;

```
## 缓存机制
在metaWithClass和classInfoWithClass中分别有2处理关于缓存的逻辑。优先会去取缓存里的数据，如果没有再进行转换然后缓存。缓存的目的是因为model类的结构通常不会动态的去修改。如果你通过runtime调整model类，调整之后需要调用setNeedUpdate去更新cache。
```
//省略部分代码
+ (instancetype)metaWithClass:(Class)cls {
    
    static CFMutableDictionaryRef cache;
    
   cache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
  
    });
}

+ (instancetype)classInfoWithClass:(Class)cls {
    static CFMutableDictionaryRef classCache;
   
    classCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
}

```

# **总结**
YYModel实现的几大关键点
1.runtime机制，操作基本都围绕着runtime展开
2.缓存机制，提高转换效率
3.提供protocol支持用户自定义匹配模式


