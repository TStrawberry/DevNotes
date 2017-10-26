# RxSwift/RxCocoa - 一种解决UITableViewCell重复订阅问题的思路

如果读者已经将RxSwift运用到项目中,或者写过相关的demo,想必一定遇到过UITableViewCell重复订阅的问题。本文将进入RxCocoa内部,粗略的探索部分源码,最终提供解决思路。



## 问题描述
UITableViewCell的点击事件可以通过订阅tableView.rx.itemSelected进行监听,但如果cell内部存在一个button1,如何订阅到该button1的点击又不会产生重复订阅的问题呢？


## 探索阶段
笔者踩坑的过程这里就不赘述了,最终的思路被定位RxCocoa对UITableView的实现上。当然,读者也可以跳过这段的内容,直接看结论。


试想一下,如果在tableView上存在一个**tableView.rx.button1Clicked**这样的实现,这不就和tableView.rx.itemSelected的方案统一了吗,所有cell上button1的点击都通过这个Observable传递出来，岂不是很nice?所以接下来我们将进入RxCocoa的源码,看看tableView.rx.itemSelected是如何实现的。

先看看tableView.rx.itemSelected实现:

```swift
/**
Reactive wrapper for `delegate` message `tableView:didSelectRowAtIndexPath:`.
*/
public var itemSelected: ControlEvent<IndexPath> {
    let source = self.delegate.methodInvoked(#selector(UITableViewDelegate.tableView(_:didSelectRowAt:)))
        .map { a in
            return try castOrThrow(IndexPath.self, a[1])
        }
    return ControlEvent(events: source)
}
```
可以看到,`self.delegate`可以监听到`tableView:didSelectRowAtIndexPath:`的点击,并且将参数转成`IndexPath`类型,所以最终我们可以tableView.rx.itemSelecte订阅到cell的点击事件。

**如果`self.delegate`也可以监听到cell上button1的点击事件,那么我就可以依葫芦画瓢的进行如下实现。**

```swift
/**
Reactive wrapper for `delegate` message `tableView:didSelectRowAtIndexPath:`.
*/
public var button1Clicked: ControlEvent<Void> {
    let source = self.delegate.methodInvoked(#selector(button1Clicked()))
        .map { _ in Void() }
    return ControlEvent(events: source)
}
```


那么我们就看看这个`self.delegate `是如何监听到`tableView:didSelectRowAtIndexPath:`的。
跳转后发现`self.delegate`的实现为:

```
extension Reactive where Base: UIScrollView {
	...
	/// Reactive wrapper for `delegate`.
	///
	/// For more information take a look at `DelegateProxyType` protocol documentation.
	public var delegate: DelegateProxy {
	    return RxScrollViewDelegateProxy.proxyForObject(base)
	}
	...
}
```
读者的第一印象可能是这个delegate是RxScrollViewDelegateProxy类型的,但是如果读者看看的RxScrollViewDelegateProxy的实现就会发现,它其实并没有遵循UITableViewDelegate协议。

**RxCocoa是Cocoa的Rx封装,所有笔者认为RxCocoa这个库在内部是不能越过Cocoa进行实现的。举个例子就是：要监听`tableView:didSelectRowAtIndexPath:`,就必须成为tableView的delegate,就必须遵循UITableViewDelegate协议。**

所以这里明显与笔者认知相悖。进而直接在RxCocoa中搜索遵循UITableViewDelegate的实现,发现只有一处结果。实现如下：

```swift
/// For more information take a look at `DelegateProxyType`.
public class RxTableViewDelegateProxy
    : RxScrollViewDelegateProxy
    , UITableViewDelegate {
    /// Typed parent object.
    public weak private(set) var tableView: UITableView?

    /// Initializes `RxTableViewDelegateProxy`
    ///
    /// - parameter parentObject: Parent object for delegate proxy.
    public required init(parentObject: AnyObject) {
        self.tableView = castOrFatalError(parentObject)
        super.init(parentObject: parentObject)
    }
}
```
可以看到RxTableViewDelegateProxy是RxScrollViewDelegateProxy的子类。同时笔者也发现另外几处有用的信息：

```
extension UIScrollView { 
    /// Factory method that enables subclasses to implement their own `delegate`.
    ///
    /// - returns: Instance of delegate proxy that wraps `delegate`.
    public func createRxDelegateProxy() -> RxScrollViewDelegateProxy {
        return RxScrollViewDelegateProxy(parentObject: self)
    }
}

extension UITableView {
    /**
    Factory method that enables subclasses to implement their own `delegate`.
    - returns: Instance of delegate proxy that wraps `delegate`.
    */
    public override func createRxDelegateProxy() -> RxScrollViewDelegateProxy {
        return RxTableViewDelegateProxy(parentObject: self)
    }
}
```
`createRxDelegateProxy`这个方法被UIScrollView实现,又被UITableView重载。其实结合上面的信息已经可以大胆的推断出:

**上面提到的`self.delegate`就是RxTableViewDelegateProxy类型的对象,它可以监听到UITableViewDelegate中定义的代理事件,也就是是说在RxCocoa内部,它就是UITableView的delegate。**

虽说是推断,但是事实也确实如此,读者可以自行验证。  
那么现在的问题就转化成了**RxTableViewDelegateProxy是如何监听到UITableViewDelegate代理事件的。**  
读者可能觉得一头雾水,不是遵循了UITableViewDelegate协议了吗,监听到代理事件就是理所因当的啊？但是有一个很重要的细节是:**RxTableViewDelegateProxy虽然遵循了UITableViewDelegate,但是它并没有实现其中的任何方法。**是不是觉得有点神奇了？

RxTableViewDelegateProxy所在继承链是这个样子的：  
```
NSObject <- _RXDelegateProxy <- DelegateProxy <- RxScrollViewDelegateProxy <- RxTableViewDelegateProxy
```
笔者在其中看到了两个关键的实现:  

```
@implementation _RXDelegateProxy
...
-(void)forwardInvocation:(NSInvocation *)anInvocation {
    BOOL isVoid = RX_is_method_signature_void(anInvocation.methodSignature);
    NSArray *arguments = nil;
    if (isVoid) {
        arguments = RX_extract_arguments(anInvocation);
        [self _sentMessage:anInvocation.selector withArguments:arguments];
    }
    
    if (self._forwardToDelegate && [self._forwardToDelegate respondsToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:self._forwardToDelegate];
    }

    if (isVoid) {
        [self _methodInvoked:anInvocation.selector withArguments:arguments];
    }
}
...
@end
```
```
open class DelegateProxy : _RXDelegateProxy {
	...
	override open func responds(to aSelector: Selector!) -> Bool {
	    return super.responds(to: aSelector)
	        || (self._forwardToDelegate?.responds(to: aSelector) ?? false)
	        || (self.voidDelegateMethodsContain(aSelector) && self.hasObservers(selector: aSelector))
	}
	...
}
```
如果读者对第一个方法名不陌生的话,相信现在你已经不觉得神奇了。反之,如果还不是太清楚,建议先看看OC消息转发机制相关的知识,然后阅读下面的内容。
  
  
**所以,至此我们就可以大致的描述`RxTableViewDelegateProxy`监听到`tableView:didSelectRowAtIndexPath:`的过程了:**
> 1. `tableView.rx.itemSelected`被执行的同时,`#selector(UITableViewDelegate.tableView(_:didSelectRowAt:))`相关的信息被保存到了DelegateProxy中。 

> 2. 当tableView的cell被点击时,会首先判断delegate是否实现了
`UITableViewDelegate.tableView(_:didSelectRowAt:)`,但是DelegateProxy重载了`responds(to:)`方法, 并且返回true, 因为第一步中已经保存了相关信息。 然后在`RxTableViewDelegateProxy`上就会执行`UITableViewDelegate.tableView(_:didSelectRowAt:)`此方法。

> 3. But, 前文已经知道`RxTableViewDelegateProxy`并没有实现该代理方法, 所以会进入消息转发的阶段。

> 4. 最终到达`_RXDelegateProxy`的 `forwardInvocation(:)`方法中。 在这里, 终于算是监听到了cell的点击事件了。既然RxTableViewDelegateProxy已经监听到了cell的点击, 剩下的就交给它就Ok, 我们订阅相应的Observable就好了。

## 最终解决方案
根据以上的结论,以下为笔者最终的解决方案(MyTableViewCell, MyTableView均为自定义view):
> * 为`MyTableViewCell`增加delegate为:
> 
```
@objc protocol MyTableViewCellProtocol: NSObjectProtocol {
    @objc optional func button1Clicked()
}
class CarsListCell: UITableViewCell {
	...
	weak var delegate: MyTableViewCellProtocol?
	func button1Clicked() {
	        if self.delegate?.responds(to: #selector(MyTableViewCellProtocol.button1Clicked())) {
            self.delegate?. button1Clicked()
        }
	}
	...
}
```
> * 扩展`RxTableViewDelegateProxy`, 同样不实现任何方法
> 
```
extension RxTableViewDelegateProxy: MyTableViewCellProtocol { }
```
> * 重写`MyTableView`的dequeue方法, 将cell的delegate设置为tableview的delegate
> 
```
class MyTableView: UITableView {
	override func dequeueReusableCell(withIdentifier identifier: String) -> UITableViewCell? {
	    let cell = super.dequeueReusableCell(withIdentifier: identifier)! as! MyTableViewCell
	    cell.delegate = self.delegate as? CarsListCellDelegate
	    return cell
	}
}
```
> * 然后扩展`Reactive<MyTableView>`
> 
```
extension Reactive where Base == MyTableView {
    var button1Clicked: ControlEvent<Void> {
        let source = self.delegate.methodInvoked(#selector(MyTableViewCellProtocol. 
        button1Clicked())).map{ _ in Void() }
        return ControlEvent(events: source)
    }
}
```
> * 现在就可以安心的使用了
> 
```
(tableView as? MyTableView).rx.button1Clicked.asObservable()
```









