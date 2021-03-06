---
layout: post
title: "Using Swift with Cocoa and Objective C 总结"
description: ""
category: swift
tags: [ios, swift]
---


### 本文是阅读Apple官方文档***Using Swift with Cocoa and Objective-C (Swift 2.2)***的总结，方便以后翻阅


#### 类方法和便捷初始化方法
##### OC的类方法转到swift中为convenience initializers



#### 可失败的初始化
##### OC中失败的初始化多返回 nil，可失败的初始化方法 可以用`nullable`标记。OC中可失败和不可失败的初始化方法转到swift表示为`init(...)` `init?(...)` 否则 为`init!(...)`



#### 存取属性
##### OC中的`@property`声明的属性转到swift中，规则如下：
- 有为空性属性的属性（`nonnull, nullable, and null_resettable`）转到swift对应于` optional 和 non-optional`类型
- OC的只读属性 在swift中表示为 计算属性 并只有一个getter（`{ get }`）
- OC的`weak`属性 在swift中用 `weak`关键字表示 （`weak var`）
- OC中的`assign, copy, strong, or unsafe_unretained` 在swift中表示为一些 适当的存储
- OC中属性的原子性(`atomic and nonatomic`) 在swift中没有对应的属性声明的映射，但是从swift中调用OC中的属性时，其原子性的实现仍然有效
- OC中的属性`getter= and setter=` 在swift中忽略




#### id的兼容性
##### swift中的`AnyObject`对应OC中的`id`，可以在`AnyObject`类型的对象上 调用任意OC的方法和属性 并且不需要类型转换，包括用`@objc`属性标记的OC兼容的属性和方法



#### Unrecognized Selectors and Optional Chaining
##### 因为`AnyObject`类型的对象的 值 到runtime时才回确定，所以很可能导致不安全的代码。跟OC一样 可能会抛出Unrecognized Selector
##### swift使用`optionals`保证代码的安全，可以像使用protocol中的可选方法一样，使用optional chaining语法 在`AnyObject`类型的对象上调用方法  
##### note：在`AnyObject`类型的对象上存取一个 属性 总是返回一个optional value。如果这个属性 正常情况下 返回optional的值，那么返回值会是双层包装的optional类型,如`AnyObject?!`
##### 虽然swift不强制解包装 `AnyObject`对象调用方法的返回值，但是为了安全编码，推荐使用解包装

{% highlight swift %}
// myObject has AnyObject type and NSDate value
let myCount = myObject.count
// myCount has Int? type and nil value
let myChar = myObject.characterAtIndex?(5)
// myChar has unichar? type and nil value
if let fifthCharacter = myObject.characterAtIndex?(5) {
    print("Found \(fifthCharacter) at index 5")
}
// conditional branch not executed
{% endhighlight %}



#### 向下转换 AnyObject
##### 当`AnyObject`类型的对象的底层数据类型已知或者可推断时，推荐向下转换到一个确定类型。因为`AnyObject`可以是任意类型，向下转换到确定类型不保证一定成功。
##### 条件类型转换操作符`as?` 返回一个optional value的目标类型。
##### 若确定是某一种类型，可使用强制类型转换符`as!`。注意 一旦强转失败 在runtime时会出错



#### Nullability and Optionals
##### OC中为初始化的指针为NULL（nil）,在swift中 ***所有类型***的值都是non-null的，并用`optional`类型去封装值缺失，用nil表示
##### OC中的Nullability可以用（`_Nullable _Nonnull nullable nonnull null_resettable`）标记，或者用`NS_ASSUME_NONNULL_BEGIN and NS_ASSUME_NONNULL_END`宏包裹某个区域。如果OC代码没有这些标记，转swift时 不能区分到底是optional还是non-optional引用 所以只能标记为隐式解封装的optional（implicitly unwrapped optional）
- `_Nonnull`标记 转为non-optional
- `_Nullable`标记 转为optional
- 没有Nullability标记声明的 转为 implicitly unwrapped optional



#### 轻量级泛型
##### OC中的Foundation框架的集合类的泛型才能转到swift中，其他类型的泛型不能转到swift 及没有泛型
{% highlight objective-c %}
@property NSArray<NSDate *> *dates;
@property NSSet<NSString *> *words;
@property NSDictionary<NSURL *, NSData *> *cachedData;

// swift
var dates: [NSDate]
var words: Set<String>
var cachedData: [NSURL: NSData]
{% endhighlight %}



#### extensions
##### swift的extensions相当于OC的category。extensions可以扩展已存在类 结构体 枚举类型 属性（属性扩展 必须为 计算属性），还可以扩展在OC中定义的这些类型。
##### extensions不能向类 结构体 枚举中添加存储属性，只能是 计算属性
##### extensions不需子类化 就可以使一个类遵守某个protocol，如果这个protocol是定义在swift中的，那么还可以使swift和OC中定义的结构体和枚举实现该protocol。
##### extensions还可以重写OC类型中 已存在的方法和属性




#### Closures

##### OC中的`block`等价与swift中的`closures`。混编时 `block`会自动转为swift中的`closures` 并标记为`@convention(block)`属性。swift中`closures`长这个样子：

{% highlight swift %}
let completionBlock: (NSData, NSError) -> Void = { data, error in
    // ...
}
{% endhighlight %}

##### `closures`跟`block`兼容，可以将swift中的`closures`传给OC中的`block`，如果一个`closures`和一个swift函数有相同的参数类型，神之可以把函数名传给OC中的`block`。  `closures`和`block`最大的区别是 后者捕获到的变量是`copy`, 而前者是`mutable`，简言之 `closures`的默认行为相当于 `block`中的捕获到用`__block`修饰的变量




#### 避免循环引用

##### OC中循环引用出现的条件:`self strong/copy block`,并且这个存在于*堆*上的`block`内部出现了`self`，使用的标准姿势如下（这可是apple的demo奥）：

{% highlight objective-c %}
__weak typeof(self) weakSelf = self;
self.block = ^{
   __strong typeof(self) strongSelf = weakSelf;
   [strongSelf doSomething];
};
{% endhighlight %}

##### swift中处理`closures`的循环引用 要讲`self`指定为`unowned`,姿势如下：

{% highlight swift %}
self.closure = { [unowned self] in
    self.doSomething()
}
{% endhighlight %}




#### 对象比较

##### swift中有两种比较操作符 `==`和`===`： 前者比较对象内容是否相同，默认调用`isEqual`方法； 后者比较对象是否指向同一个引用，检查指针是否指向同一地址。  在自定义子类中 要自己实现`isEqual`方法，这涉及到对象等同性的判断，同OC

##### note：swift自动提供equality（==）和 identity(===)操作符的逻辑实现：!= 和 !==， 这些不要重写

#### hashing

##### 这部分同OC，`NSObject`的子类在等同性判断时 也要实现`hash`属性



#### swift的类型兼容性

##### OC转swift的类中，属性 方法 下标 初始化方法等是兼容可用的，但是一些swift特有的feature是不支持的，如下：
- 泛型 generics
- 元组 tuples
- swift中的 非`Int`原始数据类型 的枚举
- swift中定义的结构体
- swift中的顶级函数
- swift中的全局变量
- swift中的类型别名 Typealiases
- swift风格的可变参数 variadics
- 嵌套类型 Nested types
- [柯里化函数](http://www.cocoachina.com/ios/20141110/10166.html)

##### swift的API转OC与OC转swift相同：
- swift中的 optional 类型标记为`__nullable`
- swift中的 non-optional 类型标记为 `__nonnull`
- swift的常量存储属性和计算属性转到OC中为read-only属性
- swift的变量存储属性转为OC中的read-write属性
- swift中的type method转为OC的类方法
- swift的初始化方法和实例方法转为OC的实例方法
- swift的抛出`errors`的方法转为OC的方法时 会在方法名后后缀一个`NSError **`类型的参数。当改swift方法没有返回值时，相应的OC方法会有一个`BOOL`类型的返回值
- ***在OC代码中，不能子类化一个swift类***

{% highlight Objective-C %}
class Jukebox: NSObject {
    var library: Set<String>
    
    var nowPlaying: String?
    var isCurrentlyPlaying: Bool {
        return nowPlaying != nil
    }
    
    init(songs: String...) {
        self.library = Set<String>(songs)
    }
    
    func playSong(named name: String) throws {
        // play song or throw an error if unavailable
    }
}

// Here’s how it’s imported by Objective-C:

@interface Jukebox : NSObject
@property (nonatomic, copy) NSSet<NSString *> * __nonnull library;
@property (nonatomic, copy) NSString * __nullable nowPlaying;
@property (nonatomic, readonly) BOOL isCurrentlyPlaying;
- (nonnull instancetype)initWithSongs:(NSArray<NSString *> * __nonnull)songs OBJC_DESIGNATED_INITIALIZER;
- (BOOL)playSong:(NSString * __nonnull)name error:(NSError * __nullable * __null_unspecified)error;
@end
{% endhighlight %}



#### 配置swift在OC中的接口

##### 在需要控制swift API向OC暴露时，可以使用`@objc(name)`属性去改变类 属性 方法 枚举类型 枚举成员暴露到OC中的名字
##### 向swift函数提供一个OC的name时，如果用的selector语法，要在selector的参数后面加`:`
##### 在swift中使用`@objc(name)`属性时，转到OC时 没有命名空间。  当把一个可归档的OC类迁移到swift时，可以使用`@objc(name)`属性指定与OC类相同的类名（因为已归档对象会存储归档的类名），这样就可以把原先已经归档的数据恢复到swift类中。
##### swift中`@nonobjc`属性 会使swift的声明在OC中无法使用。这个可以用来解决 桥接方法 的环，或者 重新加载OC导入的类中的方法。如果OC的方法被swift的方法重写 并且该方法不能暴露给OC，比如指定一个参数为变量，那么这个方法必须标记为`@nonobjc`



#### Requiring Dynamic Dispatch

##### 当在OC的runtime时 引入swift的API时，不能保证属性 方法 下标和初始化方法的动态派发。swift的编译器可能会绕过runtime 反虚拟化或者inline成员存取 来优化代码。 使用`dynamic`可以要求成员的存取方法在OC的runtime动态派发。在使用`KVO`或者OC runtime 的`method_exchangeImplementations`函数时，需要使用该修饰符。这样swift编译器inline的方法实现或者反虚拟化的存取方法不会被使用。
##### note：用`dynamic`修饰的 不能用 `@nonobjc`属性



#### OC selectors
##### OC selector在swift中用`Selector`结构体表示。构造一个selector用`#selector`表达式。如 `let mySelector = #selector(MyViewController.tappedButton)`（注意OC方法是其中的子表达式）。OC方法的引用可以括号化，可以用`as`运算符消除重载函数之间的歧义，如`let anotherSelector = #selector(((UIView.insertSubview(_:at:)) as (UIView) -> (UIView, Int) -> Void))`

##### 用performSelector发送消息
##### performSelector的方法 同步执行 并隐式返回一个`unwrapped optional unmanaged instance (Unmanaged<AnyObject>!)`，因为该方法返回值的类型和拥有关系在编译器不能确定。[ Unmanaged Objects](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/WorkingWithCocoaDataTypes.html#//apple_ref/doc/uid/TP40014216-CH6-ID85)




### 用OC的行为方式写swift类
#### 继承OC类
##### `subclassName: superclassName`  如果要重写OC中的类 要用`override`关键字


#### NSCoding
##### NSCoding协议必须实现`init(coder:)`   实现该协议的类的 子类中 如果有一个或多个***自定义的初始化方法*** 或者 其他没有初始化值的属性 也必须要重写这个方法
##### 从SB或者用`NSUserDefaults or NSKeyedArchiver`加载的对象 必须要完全实现改方法



#### 实现协议
##### OC中的协议转到swift中，会在父类后面用`,`分隔 跟随协议名称
##### 声明一个类型遵守单个协议 `id<SomeProtocol>` -> `var textFieldDelegate: UITextFieldDelegate`
##### 声明一个类型遵守多个协议 `id<SomeProtocol, AnotherProtocol>` -> `var tableViewController: protocol<UITableViewDataSource, UITableViewDelegate>`
##### note：因为在swift中 类的命名空间和协议未定义，所以OC中的`NSObject`协议映射为`NSObjectProtocol`



#### 初始化方法 和 反初始化方法
##### 












