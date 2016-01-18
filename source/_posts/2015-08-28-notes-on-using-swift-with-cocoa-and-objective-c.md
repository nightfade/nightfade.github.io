layout: post
tags: 
- iOS
- Swift
date: 2015-08-28 06:40
title: 《Using Swift with Cocoa and Objective-C》学习笔记
---

学习《[Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/)》的笔记。
# Interoperability
## Interacting with Objective-C APIs
### Initialization
在Swift中初始化一个Objective-C的类：
- Objective-C中的`init`方法要删掉`init`前缀，只是表明这个方法是一个构造器。
- `initWith`开头的要连“With”一起删除。
- 删掉`initWith`之后的`selector`分片的第一个字母改成小写。并且作为参数名。
- 不需要调用`alloc`。

```objc
UITableView *myTableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleGrouped];
```
```swift
let myTableView: UITableView = UITableView(frame: CGRectZero, style: .Grouped)
```

- Objective-C中的工厂方法被映射为Swift中的`convenience initializer`。

```objc
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```
```swift
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

#### Failable Initialization
Objective-C的构造器可以返回`nil`告知初始化失败，Swift通过可失败构造器达到相同的目的。Objective-C通过引入`Nullability and Optionals`语法表明构造器是不是可失败的，如果不可失败，Swift构造器是`init`，否则`init?`。
```swift
if let image = UIImage(contentsOfFile: "MyImage.png") {
    // loaded the imagesuccessfully
} else {
    // could not load the image
}
```

### Accessing Properties
- Objective-C中不同`nullability`特性的属性（`nonnull`，`nullable`，`null-resettable`）分别映射到Swift中的`optional`，`nonoptional`属性。
- Objective-C的`readonly`属性导入到Swift变成`{ get }`的计算属性。
- Objective-C的`weak`属性导入到Swift变成`weak var`属性。
- Objective-C的`assign`、`copy`、`strong`、`unsafe_unretained`属性导入到Swift变成相应的存储属性。
- Objective-C中的`atomic`和`nonatomic`在Swift中被忽略，全部都是`nonatomic`。
- Objective-C中的`getter=`和`setter=`在Swift中被忽略。

### Working with Methods
- Objective-C中selector的第一部分作为相应Swift方法的方法名出现在括号外。
- 剩余的selector分片变成相应的参数名出现在括号内。
- 所有的selector分片在调用的时候都是必须有的。

```objc
[myTableView insertSubview:mySubview atIndex:2];
``` 
```swift
myTableView.insertSubview(mySubview, atIndex: 2)
```
### id Compatibility
Swift有一个`AnyObject`*协议*用来表示任意类型的对象，等价于Objective-C中的`id`。可以对其调用任意Objective-C的方法和访问任意属性而不需要强制转换成特定类型。
```swift
var myObjective: AnyObject = UITableViewCell()
myObject = NSDate()

let futureDate = myObject.dateByAddingTimeInterval(10)
let timeSinceNow = myObject.timeIntervalSinceNow
```
访问`AnyObject`不存在的方法或属性会触发`runtime error`，但是可以利用Swift的可选链消除错误。另外，访问`AnyObject`的属性总是返回一个可空值。
```swift
let myChar = myObject.characterAtIndex?(5)
```
也可以强制转换`AnyObject`到一个特定类型：
```swift
let lastRefreshDate: AnyObject? = userDefaults.objectForKey("LastRefreshDate")
if let date = lastRefreshDate as? NSDate {
    // do something
}
```
### Nullability and Options
Objective-C中引用对象的指针可以是`nil`，Swift中所有的对象都是非空的，如果要表示值缺失需要用`optional`类型。
Objective-C引入了`nullability`记号来表示一个参数、属性或者返回值是否可以`nil`。
- 声明为`_Nonnull`或者包裹在`NS_ASSUME_NONNULL_BEGIN`和`NS_ASSUME_NONNULL_END`之间的类型被导入为Swift中的`non-optional`类型。
- 声明为`_Nullable`的类型被导入为Swift中的`optional`类型。
- 不带`nullability`记号的类型被导入为Swift肿的隐式`optional`类型。

```objc
@property (nullable) id nullableProperty;
@property (nonnull) id nonNullProperty;
@property id unannotatedProperty;

NS_ASSUME_NONNULL_BEGIN
- (id)returnsNonNullValue;
- (void)takesNonNullParameter:(id)value;
NS_ASSUME_NONNULL_END

- (nullable id)returnsNullableValue;
- (void)takeNullableParameter:(nullable id)value;

- (id)returnsUnannotatedValue;
- (void)takesUnannotatedParameter:(id)value;
```
导入为Swift:
```swift
var nullableProperty: AnyObject?
var nonNullProperty: AnyObject
var unannotatedProperty: AnyObject!

func returnsNonNullValue() -> AnyObject
func takesNonNullParameter(value: AnyObject)

func returnsNullableValue() -> AnyObject?
func takesNullableParameter(value: AnyObject?)

func returnsUnannotatedValue() -> AnyObject!
func takesUnannotatedParameter(value: AnyObject!)
```
### Extensions
Swift的`Extension`和Objective-C的`Category`类似。但是不能用`Extension`重写Objective-C中已经存在的方法和属性。

### Closures
Objective-C中的`block`被导入为Swift中的`closure`。
```objc
void (^completionBlock)(NSData *, NSError *) = ^(NSData *data, NSError *error) {
    // ...
}
```
等价于：
```swift
let completionBlock: (NSData, NSError) -> Void = { (data, error) in 
    // ...
}
```
可以把Swift的`closure`传给Objective-C中需要`block`的方法作为参数。

`closure`捕获变量和`block`有一点不同：Objective-C中的`__block`行为在Swift是默认的。

### Object Comparison
Swift的`==`比较运算符会调用Objective-C中`NSObject`定义的`isEqual:`方法。对于从`NSObject`继承来的类，应该实现相应的`isEqual:`方法。如果想要把对象用作字典的key，还需要实现`Hashable`协议中的`hashValue`属性。
### Swift Type Compatibility
当从Objective-C的类创建一个Swift的类的时候，它的所有成员——属性、方法、下标和构造器都是可用的。但是如果要把Swift的类（不是从Objective-C的类继承来的，或者想改变借口的名字）暴露给Objective-C，就需要显式的指定`@objc`属性。
#### Exposing Swift Interface in Objective-C
如果Swift的类是从`NSObject`或者任意Objective-C的类继承来的，那这个类就自动和Objective-C兼容。否则，用`@objc`放在Swift的方法、属性、下标、构造器，或者类、枚举的声明之前。`@objc`还能为方法指定其他名称
```swift
@objc(SomeClass)
class 中文类名: NSObject {
    @objc(initWithName:)
    init(中文参数名: String) {
        // ...
    }
    @objc(hideNuts:inTree:)
    func 中文方法(参数1: Int, 参数2: Int) {
        // ...
    }
}
```

### Lightweight Generics
```objc
@property NSArray<NSDate *> *dates;
@property NSSet<NSString *> *words;
@property NSDictionary<NSURL *, NSData *> *cachedData;
```
Swift将导入为：
```swift
var dates: [NSDate]
var words: Set<String>
var cachedData: [NSURL: NSData]
```
除了Foundation Collection的类以外，Objective-C中的lightweight generics都会被Swfit忽略。
### Objective-C Selectors
在Swift中，Objective-C的Selector被结构体`Selector`所表示。可以直接用字符串字面量构造一个Selector：`let mySelector: Selector = "tappedButton:"`。因为字符串字面量可以自动转换为Selector，所以可以把一个字符串字面量直接传给任意接收Selector参数的方法：
```swift
myButton.addTarget(self, action: "tappedButton:", forControlEvents: .TouchUpInside)
```
## Writing Swift Classes with Objective-C Behavior
### Inheriting from Objective-C Classes
Swift中可以直接定义Objective-C中类的子类。
```swift
import UIKit
class MySwiftViewController: UIViewController {
    // ...
}
```
如果要重写父类方法，需要加上`override`关键字。

### Adopting Protocols
Swift可以采用Objective-C中定义的protocol。
```swift
class MySwiftViewController: UIViewController, UITableViewDelegate, UITableViewDataSource {
    // ...
}
```
因为Swift中类的名字空间和protocol的名字空间是统一的，所以在Swift中，`NSObject`协议被映射为`NSObjectProtocol`。
### Integrating with Interface Builder
Swift中使用outlets和actions，只需要在属性前插入`@IBOutlet`或者`@IBAction`关键字。`@IBOutlet`使得属性编程隐式可选。
```swift
class MyViewController: UIViewController {
    @IBOutlet weak var button: UIButton!
    @IBOutlet var textFields: [UITextField]!
    @IBAction func buttonTapped:(_: AnyObject) {
        print("button tapped!")
    }
}
```
### Specifying Property Attributes
#### Strong and Weak
Swift属性默认是`strong`的。用`weak`关键字指示属性是弱引用，并且该属性是可选类型。
#### Read/Write and Read-Only
Swift里没有`readwrite`和`readonly`属性，如果声明存储属性，用`let`表示只读，用`var`表示可读可写。如果是计算属性，`{ get }`表示只读，`{ get set }`可读可写。
#### Copy Semantics
Swift中，Objective-C的`copy`被转换成了`@NSCopying`，也就是说该属性必须符合`NSCopying`协议。
### Using Swift Class Names with Objective-C APIs
Swift的类都放在他们所在的module的全局的命名空间里，在Objective-C引用也一样。例如，一个叫`MyFramework`的framework中的一个Swift的类`DataManager`的名字是`MyFramework.DataManager`。
```swift
let muyPersonClass: AnyClass = NSClassFromString("MyGreatApp.Person")
```

## Working with Cocoa Data Types
Swift会自动的将一些Objective-C的类型转换为Swift的类型，也会把一些Swfit类型转换为Objective-C的类型，也有一些是可以直接互换的。
### String
Swift自动的在`String`和`NSString`类型之间转换，不应该在Swift中使用`NSString`。
当Swift导入Objective-C的API的时候，会把所有的`NSString`类型替换成`String`。当Objective-C使用Swift类的时候，会把所有`String`类型替换成`NSString`。只需要`import Foundation`就可以开启两种类型的桥接。
#### Localization
Swift中用一个函数`NSLocalizedString(key:tableName:bundle:value:comment:)`替换掉了Objective-C一组宏（`NSLocalizedString`、`NSLocalizedStringFromTable`、`NSLocalizedStringFromTableInBundle`、`NSLocalizedStringWithDefaultValue`），其中为`tableName`、`bundle`、`value`都提供了默认值。
### Numbers
Swift自动把`Int`、`Float`等数值类型桥接到`NSNumber`。可以用这些数值类型创建`NSNumber`：
```swift
let n = 42
let m: NSNumber = n
```
### Collection Classes
Swift把`NSArray`、`NSSet`、`NSDictionary`分别桥接到`Array`、`Set`、`Dictionary`。
#### Arrays
如果`NSArray`指定了参数化类型，则对应到Swift中的`[ObjectType]`，否则对应到`[AnyObject]`。可以用`as?`或者`as!`把`[AnyObject]`转换到`[SomeType]`。
```swift
let swiftArray = foundationArray as [AnyObject]
if let downcastedSwiftArray = swiftArray as? [NSView] {
    // ...
}
```
如果要把Swift数组转换成`NSArray`，数组里的元素必须是符合`AnyObject`协议的。将Swift API导入Objective-C的时候，会把所有`Array`替换成`NSArray`。
#### Sets
`Set<ObjectType>`对应`NSSet<ObjectType>`，如果没有指定`ObjectType`，则对应到`Set<AnyObject>`
#### Dictionaries
`NSDictionary`如果没有指定参数化类型，则对应到`[NSObject: AnyObject]`

### Errors
Swift把`ErrorType`（协议）和`NSError`进行桥接。
```swift
@objc public enum CustomError: Int, ErrorType {
    case A, B, C
}
```
对应到相应的Objective-C的声明：
```objc
typedef SWIFT_ENUM(NSInteger, CustomError) {
    CustomErrorA = 0,
    CustomErrorB = 1,
    CustomErrorC = 2,
};
static String * const CustomErrorDomain = @"Project.CustomError"
```
### Foundation Data Types
Swift对Foundation Framework中的数据类型做了封装。
```swift
let size = CGSize(width: 20, height: 40)
let rect = CGRect(x: 50, y: 50, width: 100, height: 100)
let width = rect.width // 等价于CGRectGetWidth(rect)
let maxX = rect.maxY // 等价于CGRectGetMaxY(rect)
```
还把`NSUInteger`和`NSInteger`桥接到`Int`。
### Core Foundation
#### 重映射类型：
Swift导入Core Foundation类型的时候，编译器吧类型名中的`Ref`移除。
#### 内存管理
Swift中使用Core Foundation中`annotated`的API返回的对象会自动处理内存管理，不需要调用`CFRetain`、`CFRelase`或者`CFAutorelase`。
其他情况需要处理`Unmanaged<Instance>`。
### 错误处理
Cocoa中，会产生错误的方法把`NSError`指针作为最后一个参数，如果Objective-C方法的最后一个非block的参数是`NSError **`，Swift就把它替换成`throws`关键字，如果同时还是第一个关键字，会尝试简化方法名字，比如删除`WithError`或者`AndReturnError`后缀。如果这样的方法返回`BOOL`值表示方法成功或者失败，Swift会把返回值改成`Void`，如果返回`nil`表示失败，Swift会把返回值类型替换成`non-optional`类型。
```objc
- (BOOL)removeItemAtURL:(NSURL *)URL error:(NSError **)error;
```
```swift
func removeItemAtURL(URL: NSURL) throws
```

## Key-Value Observing
Swift中可以对继承自`NSObject`的类使用KVO，用`dynamic`关键字表明属性是需要`observe`的。
```swift
class MyObjectToObserve: NSObject {
    dynamic var myDate = NSDate()
    func updateDate() {
        myDate = NSDate()
    }
}

private var myContext = 0

class MyObserver: NSObject {
    var objectToObserve = MyObjectToObserve()
    override init() {
        super.init()
        objectToObserve.addObserver(self, forKeyPath:"myDate", options: .New, context: &myContext)
    }
    
    override func observeValueForKeyPath(keyPath:String?, ofObject object: AnyObject?, change: [String: AnyObject]?, context: UnsafeMutablePointer<Void>) {
        // ...
    }
    
    deinit {
        objectToObserve.removeObserver(self, forKeyPath:"MyDate", context: &myContext)
    }
}
```
### Singleton
```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // setup code
        return instance
    }()
}
```