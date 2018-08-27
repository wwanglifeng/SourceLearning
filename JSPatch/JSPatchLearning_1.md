# 前言
 在分析JSPatch之前，先针对JSPatch中涉及到的一些知识点做个简单的介绍。

# JavaScriptCore
## 什么是JavaScriptCore
先看下官方给的解释
> The JavaScriptCore Framework provides the ability to evaluate JavaScript programs from within Swift, Objective-C, and C-based apps. You can use also use JavaScriptCore to insert custom objects to the JavaScript environment.

简单的讲，开发者可以通过JavaScriptCore让JavaScript和native进行交互。

## 解决什么问题
解决之前native与JavaScript交互必须通过UIWebview或者WKWebView这个媒介。【可扩展】（通过UIWebView或WKWebView与js交互）

##常用的方法
先了解2个对象：
      1、  JSContext: a JavaScript execution environment
      2、  JSValue: a reference to a JavaScript value. 可以用JSValue在JS和native之间做一些基础类型的转换。【可扩展：JS和OC之间的类型转换表】
接着来了解如何使用JSContext和JSValue
```
//创建js运行环境
JSContext *context = [[JSContext alloc] init];  
//在js运行环境中执行js代码1+1，返回值是1+1执行的结果
JSValue *jsValue = [context evaluateScript:@”1 + 1”]; 
//将类型转换成OC类型
int value = jsValue.toInt32; 
//打印结果：value = 2
NSLog(@“value = %d”, value); 
                        
若JS调用OC方法，需OC注册个OC方法给JS
 context[@“_msg”] = ^(NSString *msg){
     NSLog(@“msg = %@“, msg);
  }
JS调用时也简单
_msg(‘hello world’) 即可输出日志msg = hello world
```
# runtime 的转发机制
   iOS中方法的调用实际上都是给对象发送消息，大致的流程是 
   - 通过isa指针找到对应的类
   - 在类的方法中找到对应的selector
   - 如果能找到再去调用对应的IMP

当对象接收到无法解读的消息后，就会启动“消息转发”机制。
消息转发分两大阶段：
## 动态方法解析
    
- 对象在收到无法解读的消息后，首先调用其所属类的下列方法
```          
+ (BOOL) resolveInstanceMethod:(SEL)selector;
```
    
   - 备援接受者，第一步如果没有实现，还有第二次机会处理未知的选择，在这一步中，运行期系统会问它：能不能把这条消息转给其他接受者来处理。处理方法如下：
```
+ (id) forwardingTargetForSelector:(SEL)selector;
```
## 完整的消息转发机制
- 若也没实现forwardingTargetForSelcector，就会启用完整的消息转发机制，相关的信息都放在NSInvcation对象中。  
``` 
- (void)forwardInvocation:(NSInvocation *)invocation;
```
最后附上一张《EffectiveObjective-C》中的一张图，描述了消息转发机制处理消息的各个步骤。
![图1](https://diycode.b0.upaiyun.com/photo/2018/7314012fd9f2c5a33b27d505f4ea0e54.jpg)
#NSInvocation
NSInvocation的介绍
> NSInvocation objects are used to store and forward messages between objects and between applications

再看下NSMethodSignature的介绍
> A record of the type information for the return value and parameters of a method.

在方法fowardInvocation中有唯一NSInvocation对象类型参数，我们可以从该对象中获取到参数的个数类型，返回值的类型等信息。

```
//代码来自JSPatch
NSMethodSignature *methodSignature = [invocation methodSignature];
NSInteger numberOfArguments = [methodSignature numberOfArguments];
//获取参数  返回const char *
[methodSignature getArgumentTypeAtIndex:index]; 
//获取参数  返回const char *
[methodSignature methodReturnType];       //获取返回值类型
```

我们也可以直接通过使用NSInvocation的方法来调用方法
```
  //代码来自JSPatch
 //根据selector获取到NSMethodSignature对象
[cls methodSignatureForSelector:selector];  
 //Returns an NSInvocation object able to construct message using a given method signature.
invocation = [NSInvocation invocationWithMethodSignature:methodSignature]; 
 //The receiver’s target，必须要设置
[invocation setTarget:cls]; 
//The receiver’s selector.必须要设置
[invocation setSelector:selector];
//设置参数，必须要设置
[invocation setArgument:&value atIndex:i];
//Sends the receiver;s message (with arguments) to its target and sets the return value.
//must set the receiver’s targert, selector,and argument values before calling this method.
[invocation invoke];             
```



