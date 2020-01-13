# RxSwiftDemo中的小例子源码解析



## What is Rx

[Rx](http://reactivex.io/intro.html)是一个库，提供使用可观察序列，可以组合异步和事件驱动的功能

## Rx组成

1. 可观察序列：容易创建可观察的序列
2. 操作符：可以使用操作符组合或变化可观察序列
3. 观察者：可以订阅任意流

## 使用Rx的好处

1. 基于可观察序列，使用清晰的input/output 函数 避免有状态的编程
2. Rx的操作符处理流，可以减少代码，使代码更简洁
3. 传统的Try/catch 是不支持在异步计算中的error，Rx可以支持很强的异步操作，包括处理异步的error
4. 可观察序列和调度器可以在指定的底层线程执行。支持同步和并发功能。

Rx是跨平台的

## RxSwiftDemo中的小例子源码解析

在NumbersViewController.swift中，我们可以看到一个3个输入框，然后有一个计算着3个输入框的结果。

涉及的操作符或者特有函数有**combineLatest** **rx** **text** **orEmpty** **map** **bind** **disposeBag**

Demo代码如下:

```swift
class NumbersViewController: ViewController {
    @IBOutlet weak var number1: UITextField!
    @IBOutlet weak var number2: UITextField!
    @IBOutlet weak var number3: UITextField!

    @IBOutlet weak var result: UILabel!

    override func viewDidLoad() {
        super.viewDidLoad()

        Observable.combineLatest(number1.rx.text.orEmpty, number2.rx.text.orEmpty, number3.rx.text.orEmpty) { textValue1, textValue2, textValue3 -> Int in
                return (Int(textValue1) ?? 0) + (Int(textValue2) ?? 0) + (Int(textValue3) ?? 0)
            }
            .map { $0.description }
            .bind(to: result.rx.text)
            .disposed(by: disposeBag)
    }
}
```

### **[combineLatest](http://reactivex.io/documentation/operators/combinelatest.html)**

**[combineLatest](http://reactivex.io/documentation/operators/combinelatest.html)** 是一个根据已有的几个可观察序列，创建并且返回一个新的可观察序列。当已有的可观察序列，都有发出值的时候，会捕抓每一个序列的最后一个值并且组合成元组给新的可观察序列当成元素发出。

### number1.rx

在 **Reactive.swift** 中，可以看到rx的代码

```swift
public struct Reactive<Base> {
    /// Base object to extend.
    public let base: Base

    /// Creates extensions with base object.
    ///
    /// - parameter base: Base object.
    public init(_ base: Base) {
        self.base = base
    }
}
```

```swift
extension ReactiveCompatible {
    /// Reactive extensions.
    public static var rx: Reactive<Self>.Type {
        get {
            return Reactive<Self>.self
        }
        // swiftlint:disable:next unused_setter_value
        set {
            // this enables using Reactive to "mutate" base type
        }
    }

    /// Reactive extensions.
    public var rx: Reactive<Self> {
        get {
            return Reactive(self)
        }
        // swiftlint:disable:next unused_setter_value
        set {
            // this enables using Reactive to "mutate" base object
        }
    }
}
```

从上面可知道，number1.rx是Reactive<UITextfield>一个对象。

### number1.rx.text

在 **UITextField+Rx.swift** 中可以看到大致的代码

```swift
extension Reactive where Base: UITextField {
    /// Reactive wrapper for `text` property.
    public var text: ControlProperty<String?> {
        return value
    }
    
    /// Reactive wrapper for `text` property.
    public var value: ControlProperty<String?> {
        return base.rx.controlPropertyWithDefaultEvents(
            getter: { textField in
                textField.text
            },
            setter: { textField, value in
                // This check is important because setting text value always clears control state
                // including marked text selection which is imporant for proper input 
                // when IME input method is used.
                if textField.text != value {
                    textField.text = value
                }
            }
        )
    }
}
```

这里面的getter是当rx.text作为可观察序列的时候，获取的值

这里面的setter是当rx.text作为观察者的时候，被赋值

从上面代码可以知道text是value，而value: ControlProperty<String?>是rx的controlPropertyWithDefaultEvents方法创建的一个对象。

**UIControl+Rx.swift** 的 controlPropertyWithDefaultEvents如下

```swift
extension Reactive where Base: UIControl {

    /// Creates a `ControlProperty` that is triggered by target/action pattern value updates.
    ///
    /// - parameter controlEvents: Events that trigger value update sequence elements.
    /// - parameter getter: Property value getter.
    /// - parameter setter: Property value setter.
    public func controlProperty<T>(
        editingEvents: UIControl.Event,
        getter: @escaping (Base) -> T,
        setter: @escaping (Base, T) -> Void
    ) -> ControlProperty<T> {
        let source: Observable<T> = Observable.create { [weak weakControl = base] observer in
                guard let control = weakControl else {
                    observer.on(.completed)
                    return Disposables.create()
                }

                observer.on(.next(getter(control)))

                let controlTarget = ControlTarget(control: control, controlEvents: editingEvents) { _ in
                    if let control = weakControl {
                        observer.on(.next(getter(control)))
                    }
                }
                
                return Disposables.create(with: controlTarget.dispose)
            }
            .takeUntil(deallocated)

        let bindingObserver = Binder(base, binding: setter)

        return ControlProperty<T>(values: source, valueSink: bindingObserver)
    }

    /// This is a separate method to better communicate to public consumers that
    /// an `editingEvent` needs to fire for control property to be updated.
    internal func controlPropertyWithDefaultEvents<T>(
        editingEvents: UIControl.Event = [.allEditingEvents, .valueChanged],
        getter: @escaping (Base) -> T,
        setter: @escaping (Base, T) -> Void
        ) -> ControlProperty<T> {
        return controlProperty(
            editingEvents: editingEvents,
            getter: getter,
            setter: setter
        )
    }
}
```

从上面看到使用 **Observable.create**创建一个可观察序列，

create里面先判断 weakControl，也就是UITextField自己，当自己不存在的时候，就给观察者发完成元素。

否则就从getter获取UITextField自己的当前text用next包装，并且给观察者发送下一个元素。

```swift
let controlTarget = ControlTarget(control: control, controlEvents: editingEvents) { _ in
                    if let control = weakControl {
                        observer.on(.next(getter(control)))
                    }
                }
```

上面的代码意思就是，监听control的Target-Action的editingEvents类型事件，并且给观察者发送当前的值observer.on(.next(getter(control)))

Disposables.create(with: controlTarget.dispose) 当观察者不存在，或者不需要监听了进行的一个资源释放（防止循环引用，内存泄露等）。controlTarget.dispose 主要就是remove了control的Target-Action

takeUntil(deallocated) 当前这个可观察序列，它的生命周期依赖 deallocated这个序列的生命周期。当deallocated销毁，当前的可观察序列也销毁。

let bindingObserver = Binder(base, binding: setter)

bindingObserver是一个观察者，当收到订阅消息的时候func on(_ event: Event<Element>)，它会执行setter，从而更改base的text值。**这也是为什么，number1.rx.text可以作为bind的参数**

最后，创建一个 ControlProperty<T>(values: source, valueSink: bindingObserver)，

ControlProperty对象，
当它被订阅的时候，它会订阅source，也就是订阅UITextField的值的变化（由Target-Action触发）。
当它观察别人的时候，也就是bind参数，或者subscribe参数。它会根据源的值修改自己的Text。

所以，UITextField的rx.text既是可观察序列，也是观察者。

### **number1.rx.text.orEmpty**

在ControlProperty.swift中看到 orEmpty 代码如下

```swift
extension ControlPropertyType where Element == String? {
    /// Transforms control property of type `String?` into control property of type `String`.
    public var orEmpty: ControlProperty<String> {
        let original: ControlProperty<String?> = self.asControlProperty()

        let values: Observable<String> = original._values.map { $0 ?? "" }
        let valueSink: AnyObserver<String> = original._valueSink.mapObserver { $0 }
        return ControlProperty<String>(values: values, valueSink: valueSink)
    }
}

```

```swift
let values: Observable<String> = original._values.map { $0 ?? "" }
```

从上面可以看到，它也是一层包装。转换了一下可观察序列，当字符串为nil的时候元素被转换为“”，而观察者的身份没变。

再回顾一下demo的代码：

```swift
Observable.combineLatest(number1.rx.text.orEmpty, number2.rx.text.orEmpty, number3.rx.text.orEmpty) { textValue1, textValue2, textValue3 -> Int in
                return (Int(textValue1) ?? 0) + (Int(textValue2) ?? 0) + (Int(textValue3) ?? 0)
            }
            .map { $0.description }
            .bind(to: result.rx.text)
            .disposed(by: disposeBag)
```

就可以轻易的知道大致实现了。