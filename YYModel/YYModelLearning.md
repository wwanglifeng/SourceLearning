# **前言**
 [YYModel](https://github.com/ibireme/YYModel)是ibireme写的用于模型转换的框架，可从下面几个方面去分析YYModel。
  - 文件结构：有哪些文件，每个文件大致的功能
  - 类结构及关联：有哪些类，类之间关联
  - 数据流向：debug，看数据是如何转换及看各函数之间的调用关系
  - 对重点模块进行剖析 
  - 整理源代码中自己理解不深的点（如语法、关键字的运用）

# **文件结构**
3部分，3000左右行代码
- YYModel.h 
            引入NSObject+YYModel.h 和 YYClassInfo.h
- YYClassInfo.h
           定义 YYClassIvarInfo、YYClassMethodInfo、YYClassPropertyInfo、YYClassInfo类
- NSObject+YYModel.h
            1、为NSObject、NSArray、NSDictionary写了一些category
            2、定义了_YYModelPropertMeta、_YYModelMeta

#**类结构及关联**

把主要的类及其关联画了下
![图1](https://upload-images.jianshu.io/upload_images/1477435-8d9387516e608aee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对照上图，再对各个类做一些简单的介绍
## YYClassIvarInfo
类结构中主要包含了 ivar、name等一些基本成员变量，用于保存成员变量的信息。通过以下函数进行赋值。
```
- (instancetype)initWithIvar:(Ivar)ivar;
```
## YYClassMethodInfo
类结构中主要包含SEL、IMP、name等一些基本属性，用于保存函数的信息
通过以下函数进行赋值
```
- (instancetype)initWithMethod:(Method)method;
```
## YYClassPropertyInfo
类结构中包含name、getter、setter属性，用于保存成员属性的信息。通过以下函数进行赋值
```
- (instancetype)initWithProperty:(objc_property_t)property;
```
## YYClassInfo
类结构中包含superCls、metaCls、name等一些类相关的信息。同时也将上述三个类做为类的一部分。通过以下函数进行赋值。
```
+ (nullable instancetype)classInfoWithClass:(Class)cls;
```
此函数做了2部分工作，1）判断是否有缓存，有的话就取缓存数据 2）没有缓存数据的话通过以下函数进行赋值然后缓存。
```
- (instancetype)initWithClass:(Class)cls;
```
## _YYModelPropertyMeta
类结构中主要包含YYClassPropertyInfo对象、name、mappedToKey等一些相关的信息，用来描述当前实例的属性信息。
通过以下函数进行赋值
```
+ (instancetype)metaWithClassInfo:(YYClassInfo *)classInfo propertyInfo:(YYClassPropertyInfo *)propertyInfo generic:(Class)generic;
```
## _YYModelMeta
类结构中包含了当前类的信息和不同情况下的属性数组。看定义的类结构最直观。
```
@interface _YYModelMeta : NSObject {
    YYClassInfo *_classInfo;
    /// Key:mapped key and key path, Value:_YYModelPropertyMeta.
    NSDictionary *_mapper;
       /// Array<_YYModelPropertyMeta>, all property meta of this model.
    NSArray *_allPropertyMetas;
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to a key path.
    NSArray *_keyPathPropertyMetas;
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to multi keys.
    NSArray *_multiKeysPropertyMetas;
    /// The number of mapped key (and key path), same to _mapper.count.
    NSUInteger _keyMappedCount;

   //剩下的省略。。
}
```


#**数据流向**
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
```
通过这3个函数就实现了json串到对象的转换，看似简单，其实函数中还包含不少逻辑，还是以图的形式来的实在。看下图，函数的执行顺序从左到右
![图2](https://diycode.b0.upaiyun.com/photo/2018/eb0cde2ba90041333f1f7fa7328892b5.png)

图的方式可能还有些不直观，下面把主要的代码给列出来，能更清晰的知道函数之间的调用关系。类名和函数名之后都加了test 用于区分。
```
//YYClassIvarInfoTest
@interface YYClassIvarInfoTest: NSObject
@property (nonatomic, assign, readonly) Ivar ivar;
@property (nonatomic, strong, readonly) NSString *name;

- (instancetype)initWithIvar:(Ivar)ivar;

@end

@implementation YYClassIvarInfoTest

- (instancetype)initWithIvar:(Ivar)ivar{
    _ivar = ivar;
    
    const char *name = ivar_getName(ivar);
    _name = [NSString stringWithUTF8String:name];
    
    return self;
    
}

@end


//YYClassInfoTest
@interface YYClassInfoTest: NSObject
@property (nonatomic, assign, readonly) Class cls;
@property (nullable, nonatomic, assign, readonly) Class superCls;
@property (nullable, nonatomic, assign, readonly) Class metaCls;
@property (nonatomic, readonly) BOOL isMeta;
@property (nonatomic, strong, readonly) NSString *name;

@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassIvarInfoTest *> *ivarInfos;

- (instancetype)initWithClass:(Class)cls;

@end

@implementation YYClassInfoTest

- (instancetype)initWithClass:(Class)cls{
    if (!cls) return nil;
    
    _cls = cls;
    _superCls = class_getSuperclass(cls);
    
    _isMeta = class_isMetaClass(cls);
    if(!_isMeta){
        _metaCls = objc_getMetaClass(class_getName(cls));
    }
    
    [self _update];
    
    _name = NSStringFromClass(cls);
    
    
    return self;
}

- (void)_update{
    _ivarInfos = nil;
    
    Class cls = self.cls;
    
    unsigned int ivarCount = 0;
    Ivar *ivars = class_copyIvarList(cls, &ivarCount);
    if(ivars){
        NSMutableDictionary *ivarInfos = [NSMutableDictionary new];
        _ivarInfos = ivarInfos;
        for(unsigned int i = 0; i < ivarCount; i++){
            Ivar ivar = ivars[i];
            
            YYClassIvarInfoTest *info = [[YYClassIvarInfoTest alloc] initWithIvar:ivar];
            if(info.name){
                ivarInfos[info.name] = info;
            }
        }
                
        free(ivars);
    }
    
}

+ (instancetype)metaWithClass:(Class)cls{
    //是否有缓存判断
    YYClassInfoTest *meta = [[YYClassInfoTest alloc] initWithClass:cls];
    return meta;
}

@end


///_YYModelPropertyMetaTest
@interface _YYModelPropertyMetaTest: NSObject

@end
@implementation _YYModelPropertyMetaTest

@end




///_YYModelMetaTest
@interface _YYModelMetaTest: NSObject{
    @package
    YYClassInfoTest *_classInfoTest;
    NSDictionary *_mapper;
}

@end

@implementation _YYModelMetaTest

- (instancetype)initWithClass:(Class)cls{
    YYClassInfoTest *classInfoTest = [YYClassInfoTest metaWithClass:cls];
    _classInfoTest = classInfoTest;
    
   
    
    return self;
}

+ (instancetype)metaWithClass:(Class)cls{
    //是否有缓存判断
    _YYModelMetaTest *meta = [[_YYModelMetaTest alloc] initWithClass:cls];
    return meta;
}

@end

typedef struct {
    void *modelMeta;  ///< _YYModelMeta
    void *model;      ///< id (self)
    void *dictionary; ///< NSDictionary (json)
} ModelSetContextTest;

static void ModelSetWithDictionaryTestFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContextTest *context = _context;
    __unsafe_unretained _YYModelMetaTest *meta = (__bridge _YYModelMetaTest *)(context->modelMeta);
    __unsafe_unretained _YYModelPropertyMetaTest *propertyMeta = [meta->_mapper objectForKey:(__bridge id)(_key)];
    __unsafe_unretained id model = (__bridge id)(context->model);
    while (propertyMeta) {
        if (propertyMeta->_setter) {
            
            //ModelSetValueForProperty函数的核心是调用objc_msgSend进行赋值
            ModelSetValueForProperty(model, (__bridge __unsafe_unretained id)_value, propertyMeta);
            
        }

    };
}


///YYModelTest
@interface NSObject (YYModelTest)

+ (nullable instancetype)yy_modelWithJSONTest:(id) json;

@end

@implementation NSObject (YYModelTest)

- (BOOL)yy_modelSetWithDictionary:(NSDictionary *)dic {
    _YYModelMetaTest *modelMeta = [_YYModelMetaTest metaWithClass:object_getClass(self)];
    
    ModelSetContextTest context = {0};
    context.modelMeta = (__bridge void *)(modelMeta);
    context.model = (__bridge void *)(self);
    context.dictionary = (__bridge void *)(dic);
    
    CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryTestFunction, &context);

    return YES;
}

+ (instancetype)yy_modelWithJSONTest:(id)json {
    
    Class cls = [self class];
    _YYModelMetaTest *modelMeta = [_YYModelMetaTest metaWithClass:cls];
    
    NSObject *one = [cls new];
    if ([one yy_modelSetWithDictionary:json]) return one;
    return nil;
    
}


@end


```

## yy_modelToJSONObject
该函数的流程相对简单点，从_YYModelMeta获取相关的数据赋值给NSDictionary对象。不再做扩展。


#**重点功能剖析**
##键值的匹配
如果不用第三方框架，自己写代码来做转换，最简单方式如下
```
//省略部分代码
 User *user = [[User alloc] init];
 user.name = json[@"name"];
```
这个方式在类属性较多的情况下，需要手动写很多相似代码，比如
```
User *user = [[User alloc] init];
user.name = json[@"name"];
user.birthday = json[@"birthday"];
user.sex = json[@"sex"];
user.age = json[@"age"];
```
极端的情况下，如果有100个属性，那相似代码就会更多。有一种比较好的方式是通过runtime机制，通过循环对对象属性赋值。
YYModel也是基于runtime机制去实现模型转换，YYModel除了支持单个键值匹配，还支持一些特定的匹配

##缓存机制


###类结构定义的目的
为什么这么定义：跟runtime提供的接口有关
