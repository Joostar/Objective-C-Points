# Objective-C-Points
标签： Objetive-C
---
记录objc学习中的一些技术点。

---
[TOC]

----
###1.内存管理，各种属性关键字意义、区别
---
####nonatomic
非原子操作，没有线程锁，效率高

---
####atomic
默认属性关键字之一，原子操作，由于系统给加的线程锁，效率低。只能保证`set` `get`方法的线程安全，不能保证对于成员变量访问的绝对安全和数据的一致性，比如在一个线程 *thread1* 内进行：
``` objective-c
//对于某个对象obj,存在属性@proprety(atomic,readwrite) int i;
for(int i = 0;i < 1000;i++)
{
NSLog(@"%d",obj.i);
}
```
而存在另外一个并发执行的线程 *thread2* ，在某个时刻执行:
```objective-c
obj.i = 10;//set
```
这可能会导致 *thread1* 输出不一致。

而有些情况，如：
```objective-c
@interface SomeClass : NSObject
@property (atomic, retain) NSMutableArray *mArray;
@end 
```
同时存在某个线程 *thread1* 、*thread2* 对于`SomeClass`的实例 *obj* 执行如下操作：
```objective-c
for(int i = 0;i < 1000;i++)
{
[obj.mArray addObject:[NSNumber numberWithInt:i]];
}
@end 
```
将可能会出现多线程访问冲突。
综上：**atomic** 并不能绝对保证多线程的安全。

---
####readonly
只读权限，默认不会合成`set`方法。

---
####readwrite
可读写权限。
如果不显示声明则**默认为readwrite**。

---
####assign
可以用于**C数据类型**以及**类类型**对象，作用于**类类型**对象时，不会增加引用计数，目标对象释放后**不会置为nil**。

---
####unsafe_unretained
作用同**assign**。

---
####retain
对原对象`release`,对目标对象`retain`。只能作用于**类类型**对象，会增加引用计数而**强引用**到目标对象。

---
####strong
作用同**retain**，`ARC`下默认关键字之一。

---
####copy
对原对象`release`,对目标对象进行`copy`操作并指向copy后指向`copy`后的地址。不能作用于**C数据类型**，目标对象必须实现`NSCopying`协议。

---
####weak
在`ARC`下使用，对目标对象进行**弱引用**，当目标对象释放后**会置为nil**。

---
####说明及注意

---
`MRC`下默认关键字：`atomic` `readwrite` `assign`
`ARC`下默认关键字：`atomic` `readwrite` `strong`

---
在`ARC`环境下，对于**变量**可以用`__weak` `__strong` `__unsafe_unretained` `__autoreleasing`来修饰以控制变量是否进行**强引用**等行为。而默认值是`__strong`。例如：
```objective-c
NSObject * obj = [NSObject alloc] init];
//
NSObject * obj1 = obj;
__strong NSObject * obj2 = obj;//
__weak NSObject * objc3 = obj;
```
*obj1* 与 *obj2* 都会对 *obj* 进行**强引用**，而 *objc3* 为**弱引用**。

`__autoreleasing`关键字对用于函数返回值，当需要返回某个对象时，系统会默认给加上。

需要注意的是，`ARC`中的**成员变量**其默认修饰关键字为`__strong`,所以如下写法会出现**编译错误**：
```objective-c
@interface SomeClass : NSObject
{
NSObject * _obj;
}
@property (nonatomic, weak) NSObject *obj;
@end 

@implementation SomeClass
@end
```

---
声明为`(nonatomic,copy)`的属性，在调用`set`方法时，会调用`objc_setProperty_nonatomic_copy`,然后调用`copyWithZone`方法。
声明为`(atomic,copy)`的属性，在调用`set`方法时，会调用`objc_setProperty_atomic_copy`,然后调用`copyWithZone`方法。

----
####对于`copy`（`mutableCopy`）的进一步说明

---
**NSObject**类中存在一些方法：
```objective-c
+(instancetype)alloc;
+ (instancetype)allocWithZone:(struct _NSZone *)zone;

- (id)copy;
- (id)mutableCopy;

+ (id)copyWithZone:(struct _NSZone *)zone;
+ (id)mutableCopyWithZone:(struct _NSZone *)zone;
```
其中后两个方法是为`类对象`服务的，以便让`类对象`存放到容器中，但`类对象`全局只需要**一份**，所以只需要直接返回`self`,如下：
```objective-c
+ (id)copyWithZone:(struct _NSZone *)zone
{
return self;
}
+ (id)mutableCopyWithZone:(struct _NSZone *)zone
{
return self;
}
```
对于`+(instancetype)alloc` 方法，将直接返回`+ (instancetype)allocWithZone:(struct _NSZone *)zone`的返回值，由于**NSZone**已经被废弃，所以直接传入`NULL`，如下：
```objective-c
+ (id)alloc
{
return [self allocWithZone:NULL];
}
```
对于`- (id)copy`方法，将直接返回`NSCopying`协议中的`- (id)copyWithZone:(nullable NSZone *)zone`的返回值，如下：
```objective-c
- (id)copy
{
return [self copyWithZone:NULL];
}
```
但NSObject并没有实现`- (id)copyWithZone:(nullable NSZone *)zone`方法，所以如果需要`copy`则需要子类实现此方法。

**同理**对于`- (id)mutableCopy`方法，则需要实现`NSMutableCopying`中的`- (id)mutableCopyWithZone:(nullable NSZone *)zone`方法。

---
对于
```objective-c
NSArray * array = [NSArray arrayWithObjects:/*some objects*/, nil];

NSMutableArray * mArray = [NSMutableArray array];
[mArray addObject:/*some object*/];
```
会对目标对象进行`retain`操作，而不是 `copy`操作。

---
对系统**容器类**进行`copy`或者`mutableCopy`的说明：
以下以**Array**类为例，对于**Dictionary** 或者 **Set** 其行为类似。

---
对**NSArray** 的实例对象 *array*进行`copy`操作：
返回值为`[array retain]`后的地址，及只会对*array*进行`retain`,而不会重新分配内存。另外 *array* 中的所有元素**没有**进行`retain`操作。

对**NSArray** 的实例对象 *array2*进行`mutableCopy`操作：
返回值为**NSMutableArray**类对象，地址与*array2* **不同**，`retainCount`为`1`,并且对*array2*中的所有元素**进行**`retain`（注意不是`copy`或者`mutableCopy`）操作。

---
对**NSMutableArray** 的实例对象 *mArray*进行`copy`操作：
返回值为**NSArray**类对象，地址与*mArray* **不同**，`retainCount`为`1`,并且对*mArray*中的所有元素**进行**`retain`操作。

对**NSMutableArray** 的实例对象 *mArray2* 进行`mutableCopy`操作：
返回值为**NSMutableArray**类对象，地址与*mArray2* **不同**，`retainCount`为`1`,并且对*mArray2* 中的所有元素**进行**`retain`操作。

---
**在实际开发过程中：**

1. 对于祖先链上**无** `- (id)copyWithZone:(nullable NSZone *)zone`方法的，直接调用
```objective-c
- (id)copyWithZone:(nullable NSZone *)zone
{
id res = [[self class] allocWithZone:zone];
//对其他成员变量复制或者赋值
//
return res;
}
```
注意这里用`[self class]`而**不是**用具体的类名，否则**子类**再**重写**此方法时会有问题：`alloc`出来的会是**父类**类型而**不是子类**类型，**不会**包含**子类**中的成员变量等。
2. 对于祖先链上有`- (id)copyWithZone:(nullable NSZone *)zone`方法的，直接调用：
```objective-c
- (id)copyWithZone:(nullable NSZone *)zone
{
id res = [super copyWithZone:zone];
//对本类的其他成员变量复制或者赋值
//
return res;
}
```
先调用**父类**中的方法，然后对**本类**中的成员变量处理。
3. 对于**非mutable**对象，仅仅对本对象`retain`即可，当然也是根据自己需求灵活调整。
