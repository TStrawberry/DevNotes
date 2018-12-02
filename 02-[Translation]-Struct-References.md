# 为结构体添加引用语义 

> 注：[原文地址](http://chris.eidhof.nl/post/references/)

> [点击链接](https://gist.github.com/chriseidhof/3423e722d1da4e8cce7cfdf85f026ef7)查看示例代码.

最近一段时间,我一直在探索新的Swift KeyPath特性,并且有了一些收获。文中提到的内容都还只是实验性的,并未在生产环境验证过其有效性。也就是说我只是觉得这很酷, 所以分享出来。

我们以一个简单的通讯地址app为例,其中包含了一个展示所有联系人的tableView和一个展示联系人 Person 详细信息的viewController。  

如果将 Person 和 Address 定义成一个class, 那么它的信息大致是这样的:  

```swift
class Person {
    var name: String
    var addresses: [Address]
    init(name: String, addresses: [Address]) {
        self.name = name
        self.addresses = addresses
    }
}

class Address {
    var street: String
    init(street: String) {
        self.street = street
    }
}

```
而 PersonVC 包含一个 person 属性和一个用于修改 person 的 change 方法:  

```swift
final class PersonVC {
    var person: Person
    init(person: Person) {
        self.person = person
    }
    
    func change() {
        person.name = "New Name"
    }
}
```
将 Person 定义成class会出现诸多问题:  

* `person`是一个引用, 这意味着它可以同时具备多个持有者, 因而可以在程序的多处被修改。诚然,这可以让我们方便地进行程序间通信。但于此同时, 当这些修改发生时, 我们也需要及时被告知(例如使用KVO),否则将无法保证数据修改和界面显示的同步。目前来说, 这并不容易。  
* 监听嵌套的对象属性已经不易,对 Person.addresses 上的改变进行监听更是难上加难。  
* 如果我们需要一个person的拷贝,就需要实现 NSCopying 协议。这带来的不仅仅是工作量的增加,还需要考虑深浅拷贝的问题。  
* 如果多个 Person 对象存在于一个数组中,而我们希望知道这个数组何时改变(例如:序列化)了。想要做到这些要么需要大量的样板代码要么需要大量的监听动作。

但是将 Person 和 Address 定义成 结构体 也并不会让问题变得简单多少:  

* 每个结构体实例都是一个独立的拷贝。它始终保持着简单的持有关系,并且所有的修改行为我们都十分清楚。但是当我们在 PersonVC 中修改了person后,问题就开始显现了。这些修改无法被tableView获得,自然也无法更新对应的界面。如果是对象,这一切就会是顺理成章的。
* 退一步讲, 即使我们能够实现对 Person 数组的修改行为进行监听, 也无法对某个单一对象上的修改进行监听(例如: 对数组中第一个 Person 实例的name属性的修改行为)。

而综合这两种方案的优点便是最终的方案了:

* 拥有可变共享引用。
* 将底层数据设计成结构体来方便的获得独立的拷贝。
* 可以对任意层级上的修改进行监听。(例如上面提到的: 对数组中第一个 Person 实例的name属性的修改行为)。

接下来我将介绍它的使用方法, 工作原理以及目前存在的限制。  

## 使用方法  
首先使用结构体定义Addressbook:  

```swift
struct Address {
    var street: String
}
struct Person {
    var name: String
    var addresses: [Address]
}

typealias Addressbook = [Person]
```
现在, 我们就可以使用Ref类型(Reference的简写)了。我们创建一个新的addressBook, 同时用一个空数组初始化它。然后添加一个新的 ***Person*** 实例。接下来就是炫酷的地方了: 通过使用下标脚本, 我们可以获得对第一个 Person 的**引用**以及对它的名字的**引用**。尝试将它的名字修改成“New Name”, 然后验证修改是否被添加到了 addressBook 中:  

```swift
let addressBook = Ref<Addressbook>(initialValue: [])
addressBook.value.append(Person(name: "Test", addresses: []))
let firstPerson: Ref<Person> = addressBook[0]
let nameOfFirstPerson: Ref<String> = firstPerson[\.name]
nameOfFirstPerson.value = "New Name"
addressBook.value // shows [Person(name: "New Name", addresses: [])]
```
firstPerson和nameOfFirstPerson的类型可以省略, 这里只是为了代码的可读性。  
我们可以在需要的地方获得一个 Person 的独立拷贝。使用 myOwnCopy 获得一个独立拷贝, 并且不需要实现 NSCopying 协议:  

```swift
var myOwnCopy: Person = firstPerson.value
```
同时, 我们也可以像使用一些响应式框架那样对Ref进行监听, 然后得到一个类似的 disposable 实例来管理监听者的生命周期:  

```swift
var disposable: Any?
disposable = addressBook.addObserver { newValue in
    print(newValue) // Prints the entire address book
}

disposable = nil // stop observing
```
监听 nameOfFirstPerson 也是一样的。  

```swift
nameOfFirstPerson.addObserver { newValue in
    print(newValue) // Prints a string
}
```
截至目前, 我们的监听对象还局限在 Addressbook 中, 更多内容接下来会逐一介绍。

目光回到 PersonVC 中。使用 Ref 替换相关实现后, 控制器中可以对 person 进行监听了。在响应式编程中, 一个信号被设计成read-only, 它只能监听值的改变, 你需要其他的方式修改它。 但是在 Ref 的实现中, 使用 person.value 就可以轻松的完成:  

```swift
final class PersonVC {
    let person: Ref<Person>
    var disposeBag: Any?
    init(person: Ref<Person>) {
        self.person = person
        disposeBag = person.addObserver { newValue in
            print("update view for new person value: \(newValue)")
        }
    }
    
    func change() {
        person.value.name = "New Name"
    }
}
```
PersonVC 并不关心 Ref<Person> 来自何处, 可能是一个 Person 数组,数据库亦或者其他情况。通过将数组封装到一个 History 结构体中, 我们可以为 Addressbook 添加undo支持, 并且不用修改任何 PersonVC 中的代码:

```swift
let source: Ref<History<Addressbook>> = Ref(initialValue: History(initialValue: []))
let addressBook: Ref<Addressbook> = source[\.value]
addressBook.value.append(Person(name: "Test", addresses: []))
addressBook[0].value.name = "New Name"
print(addressBook[0].value)
source.value.undo()
print(addressBook[0].value)
source.value.redo()
```
实际上,很多有用的特性都可以被添加到Ref的实现中: caching, serialization, automatic synchronization 等等, 不过这些内容超出了本文的讨论范围。

## 实现细节  
下面看看这一切是如何实现的？我们将从 Ref 开始。Ref 的实现包含以下几个部分: 如何对底层的值进行读写, 如何添加监听者。它拥有一个构造器要求这三者的信息: 

```swift
final class Ref<A> {
    typealias Observer = (A) -> ()
    
    private let _get: () -> A
    private let _set: (A) -> ()
    private let _addObserver: (@escaping Observer) -> Disposable
    
    var value: A {
        get {
            return _get()
        }
        set {
            _set(newValue)
        }
    }
    
    init(get: @escaping () -> A, set: @escaping (A) -> (), addObserver: @escaping (@escaping Observer) -> Disposable) {
        _get = get
        _set = set
        _addObserver = addObserver
    }    

    func addObserver(observer: @escaping Observer) -> Disposable {
        return _addObserver(observer)
    }
}
```
接下来我们就为其添一个便利构造器用于监听一个结构体值。在该构造器中创建了一个保存observers和variable的字典。当variable改变时, 所有的 observers 都会被通知。它调用了上面定义的指定构造器。

```swift
extension Ref {
    convenience init(initialValue: A) {
        var observers: [Int: Observer] = [:]
        var theValue = initialValue {
            didSet { observers.values.forEach { $0(theValue) } }
        }
        var freshId = (Int.min...).makeIterator()
        let get = { theValue }
        let set = { newValue in theValue = newValue }
        let addObserver = { (newObserver: @escaping Observer) -> Disposable in
            let id = freshId.next()!
            observers[id] = newObserver
            return Disposable {
                observers[id] = nil
            }
        }
        self.init(get: get, set: set, addObserver: addObserver)
    }
}
```
考虑一下 Person 的引用, 为了得到一个对其 name 属性的引用, 我们需要一种可以读写其 name 属性的方式, WritableKeyPath 正是我们所需要的。我们可以为 Ref 添加 下标脚本 用于获得一个 Person 内部值的引用:

```swift
extension Ref {
    subscript<B>(keyPath: WritableKeyPath<A,B>) -> Ref<B> {
        let parent = self
        return Ref<B>(get: { parent._get()[keyPath: keyPath] }, set: {
            var oldValue = parent.value
            oldValue[keyPath: keyPath] = $0
            parent._set(oldValue)
        }, addObserver: { observer in
            parent.addObserver { observer($0[keyPath: keyPath]) }
        })
    }
}
```
上面的代码理解起来可能存在一定难度, 但是如果单纯为了使用, 你其实也不必纠结它的实现。
终有一天, KeyPath 将支持下标脚本特性。但目前, 我们还是必须自己为集合类型添加这个实现。具体代码根上面的很相似, 唯一的区别是我们使用了索引而非 KeyPath: 

```swift
extension Ref where A: MutableCollection {
    subscript(index: A.Index) -> Ref<A.Element> {
        return Ref<A.Element>(get: { self._get()[index] }, set: { newValue in
            var old = self.value
            old[index] = newValue
            self._set(old)
        }, addObserver: { observer in
                self.addObserver { observer($0[index]) }
        })
    }
}
```
以上就是所有的实现了。虽然使用了很多 Swift 的高级特性, 但是也让代码保证在了百行内。 其中也不乏一些 Swift 4 添加的新特性: keypaths, generic subscripts, open-ended ranges等, 甚至很多特性目前你只能在 Swift 这门语言中见到。

## 讨论  
如同一开始所说的, 这些都是实验性的代码, 如果将其运用到真实的app中不知效果如何, 我对其抱有极大兴趣。例如下面这个代码片段对我来说是违反直觉的:

```swift
var twoPeople: Ref<Addressbook> = Ref(initialValue:
    [Person(name: "One", addresses: []),
     Person(name: "Two", addresses: [])])
let p0 = twoPeople[0]
twoPeople.value.removeFirst()
print(p0.value) // what does this print?
```
我也想更进一步为其添加更多功能。例如队列的支持, 你就可能这样调用了:

```swift
var source = Ref<Addressbook>(initialValue: [], 
    queue: DispatchQueue(label: "private queue"))
```
将其整合进数据库也是不错的想法。`Var` 意味着你可以进行读写操作, 已经订阅到任何通知流程中去:

```swift
final class MyDatabase {
   func readPerson(id: Person.Id) -> Var<Person> {
   }
}
```
非常期待收到任何意见和反馈。尝试自己去实现一下你将对其中原理更加了解。顺便提一句, 我们将制作两期 [Swift Talk](https://talk.objc.io) 专门讲解以上内容。如果你对 Florian 和我的探索过程感兴趣, 可以订阅该节目。

