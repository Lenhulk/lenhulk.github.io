---
title: 解决scrollView中恼人的手势冲突
date: 2019-12-19 23:39:34
tags: [iOS,开发之术,ScrollView,TableView,手势]
Categories: iOS
---

{% cq %}

开发一个界面较为复杂的app，我们势必会出现在scrollview或者tableview上加上按钮或者手势，But..

Cell中Button点击高亮反馈为何有阻滞？

又是什么阻挡了我tableview的滑动（滑动失效）？

{% endcq %}

<!-- more -->



# ScrollView手势处理原理

一款优秀的产品势必要有优秀的点击反馈，但是为何你做的列表 **快速点击了cell上的按钮没有高亮，点击时间延长了才有效果！？**

> UIScrollView原理，以时间为轴线：
>
> 从你的手指touch屏幕开始，scrollView开始一个timer，如果：
>
> 1. 150ms内如果你的手指没有任何动作，消息就会传给subView。
>
> 2. 150ms内手指有明显的滑动（一个swipe动作），scrollView就会滚动，消息不会传给subView。
>
> 3. 150ms内手指没有滑动，scrollView将消息传给subView，但是之后手指开始滑动，scrollView传送touchesCancelled消息给subView，然后开始滚动。
>
>    
>
> UIScrollView有一个BOOL类型的`tracking`属性，用来返回用户是否已经触及内容并打算开始滚动,当手指触摸到UIScrollView内容的一瞬间，会产生下面的动作：
>
> 拦截触摸事件
>
> tracking属性变为YES
>
> 一个内置的计时器开始生效，用来监控在极短的事件间隔内是否发生了手指移动
>
> * 当检测到时间间隔内手指发生了移动，UIScrollView自己触发滚动，tracking属性变为NO，手指触摸下即使有(可以响应触摸事件的)内部控件也不会再响应触摸事件。
>
> * 当检测到时间间隔内手指没有移动，tracking属性保持YES，手指触摸下如果有(可以响应触摸事件的)内部控件，则将触摸事件传递给控件进行处理。

当你手指放在按钮上，维持150ms，才会触发UIButton的触摸事件！





### delaysContentTouches

不过上面的工作原理其实有一个属性开关来控制：`delaysContentTouches`（默认true）

只要将scrollview的这个属性设置为false，滚动视图将会第一时间处理响应者链的手势传递！

（按钮一点即亮～）

这时就会立刻执行`touchesShouldBegin: withEvent: inContentView: `方法！

（这个方法是最先接收到滑动事件的，优先于button的

`UIControlEventTouchDown`，以及`- (void)touchesCancelled:(NSSet*)touches withEvent:(UIEvent*)event）`

> 如果设置为true，touch事件沿着消息响应链传递;

> 如果设置为false，表示UIScrollView接收这个滚动事件，不必沿着消息响应链传递了。





### touchesShouldCancelled:withEvent:

当我们正在触摸屏幕的时候，如果出现了低电量、有电话呼入等等这样的系统事件时候，低电量或者电话的窗口会置为前台，这个时候touchesCancelled方法就会被调用。这大多数是由iOS系统发出的一些事件，导致触摸事件的中断，一般情况下直接调用touchesEnd即可。

> 如果返回true  ，(系统默认)是允许UIScrollView，按照消息响应链向子视图传递消息的

> 如果返回false，UIScrollView,就接收不到滑动事件了。





### canCancelContentTouches

这个Bool类型的值控制contentview里的触摸是否总能引发跟踪(tracking)。（默认true）

如果设置为false，这消息一旦传递给subView，这scroll事件不会再发生。

> 如果属性值为true并且跟踪到手指正触摸到一个内容控件，这时如果用户拖动手指的距离足够产生滚动，那么内容控件将收到一个`touchesCancelled:withEvent:`消息，而scrollview将这次触摸作为滚动来处理。

> 如果值为false，一旦contentview开始跟踪(tracking==YES)，则无论手指是否移动，scrollView都不会滚动。







# 将按钮点击反馈出来

为了使我的cell或者子控件上的按钮有点击效果，将 `delaysContentTouches`设置为false，我的按钮重获了第一时间的反馈效果。

当`delaysContentTouches`设置为false之后，scrollview的滑动有时会失效， 我的手指经过了button（有点击高亮效果）再移开，scrollview不会接收滑动事件，UIButton阻挡掉了我的滑动手势！

如果同时再主动修改`canCancelContentTouches`这个属性改为true，滑动手势仍然会被阻挡😂（这个属性基本没有用到过，但是先记一下）。







# 修复滑动手势

**效果：**当将手指按下时，cell上的按钮会更改颜色，这会提供视觉反馈，以确认您抬起手指时将激活哪个按钮。接着用户如果将手指从按钮上拖动，则划出该按钮时，触摸这个事件将被取消，并且该按钮取消高亮显示，将用户手势变为滑动滚动scrollview。



这时我们的技巧是，重写上文中提到的，`touchesShouldCancel(in:)` ，自己决定是否允许滚动视图取消控件中的触摸！



这里实现了一个scrollView, tableView, collectionView的父控件，他们可以防止UIButton或者某些不应该吃掉手势的控件作出不当的行为（但是最好还是保留UITextInput, UISlider, 以及UISwitch阻挡手势的权利）：

```swift
import UIKit

// These let you start a touch on a control that's inside a scroll view,
// and then if you start dragging, it cancels the touch on the button
// and lets you scroll instead. Without these scroll view subclasses,
// controls in scroll views will eat touches that start in them, which
// prevents scrolling and makes the app feel broken.
//
// The UITextInput exception is for cases where you have a text field
// or a text view in a scroll view. If you press and hold there, you want
// to get the text editing magnifier cursor, instead of canceling the
// touch in the text input element.
//
// Ditto for UISlider and UISwitch: if the table view eats the drag gesture,
// they feel broken. Feel free to add your own exceptions if you have custom
// controls that require swiping or dragging to function.

final class ControlContainableScrollView: UIScrollView {

    override func touchesShouldCancel(in view: UIView) -> Bool {
        if view is UIControl
            && !(view is UITextInput)
            && !(view is UISlider)
            && !(view is UISwitch) {
            return true
        }

        return super.touchesShouldCancel(in: view)
    }

}

final class ControlContainableTableView: UITableView {

    override func touchesShouldCancel(in view: UIView) -> Bool {
        if view is UIControl
            && !(view is UITextInput)
            && !(view is UISlider)
            && !(view is UISwitch) {
            return true
        }

        return super.touchesShouldCancel(in: view)
    }

}

final class ControlContainableCollectionView: UICollectionView {

    override func touchesShouldCancel(in view: UIView) -> Bool {
        if view is UIControl
            && !(view is UITextInput)
            && !(view is UISlider)
            && !(view is UISwitch) {
            return true
        }

        return super.touchesShouldCancel(in: view)
    }

}
```



只要使用了这个两个技巧，你能保留scrollview上交互控件的良好点击反馈，又能像原生tableview一样自然的滑动取消点击cell的高亮并且变成滚动scrollview，带来更自然更好的用户体验哇！！！







# UIGestureRecognizerDelegate解决scrollView嵌套问题

—— 当你需要在scrollView上加上滑动手势的时候，或者遇到scrollView嵌套scrollView的时候。

`*gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)*`

这个方法可以让手势继续向下一个响应者传递，手势加在ScrollView或者tableView上也**不会**有冲突，不需要通过offset差值去计算滑动方向，效果相当完美。



**比方有这样一个普通需求：**

1. 上滑的时候，先不让子scrollView滑动（更改contentOffsetY为0），而是隐藏`topView`头部内容控件，再让子scrollView开始滑动

2. 下滑的时候，等子scrollView滑到最上方之后（contentOffsetY为0），再允许父ScrollView滑动，滚动到`topView`头部控件顶部

   

**我们可以这样实现：**

1. 创建UIScrollView的子类，实现代理UIGestureRecognizerDelegate的`gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)`方法；返回true，让滑动事件可以向父View传递：

```swift
class MyScrollView: UIScrollView, UIGestureRecognizerDelegate {
  
  var isSimultaneousGesture: bool = true
  
	func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer, shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer) -> Bool {
		return self.isSimultaneousGesture
	}
}
```

2. 实现代理方法：（例子）

```swift
extension CollisionViewController: UIScrollViewDelegate {
	func scrollViewWillBeginDragging(_ scrollView: UIScrollView) {
		if scrollView == self.myScrollView {
      // 记录子scrollview的初始y轴位移
			_lastOffsetY = self.myScrollView.contentOffset.y
		}
	}
  
	func scrollViewDidScroll(_ scrollView: UIScrollView) {
		if scrollView == self.myScrollView {
      // anchor反正就是topView的底部距离，设为一个产生形变的锚点
			let anchorY = <topView>.bottom
      
			if self.scrollView.contentOffset.y > _lastOffsetY {
				// scrollView下滑
				if superContainerScrollView!.contentOffset.y <= anchorY {
					self.scrollView.contentOffset.y = 0
          superContainerScrollView?.setContentOffset(CGPoint(0, anchorY) animated: true)
          //让手势透传属性变为false
          scrollView.isSimultaneousGesture = false
          //这里可以做隐藏navigationBar的操作..
				}
        
			} else if self.scrollView.contentOffset.y < _lastOffsetY {
				// scrollView上滑
				if self.scrollView.contentOffset.y < 0 {
          self.scrollView.contentOffset.y = 0
					superContainerScrollView?.setContentOffset(CGPoint(0, 0) animated: true)
          //让手势透传属性变为true
          scrollView.isSimultaneousGesture = true
          //这里可以做显示navigationBar的操作..
				}
			}
		}
	}
}
```





