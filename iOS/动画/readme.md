

# 动画

## iOS 动画的结构





## 基础动画（BasicAnimation）



## 关键帧动画



## 动画组





## UIView动画







transition介绍

When you add the transition as an animation, an implicit CATransaction is begun. From that point on, all modifications to layer properties are going to be animated rather than immediately applied. The way the CATransition performs this animation to to take a snapshot of the view before the layer properties are changed, and a snapshot of what the view will look like after the layer properties are changed. It then uses a filter (on Mac this is Core Image, but on iPhone I'm guessing it's just hard-coded math) to iterate between those two images over time.

This is a key feature of Core Animation. Your draw logic doesn't generally need to deal with the animation. You're given a graphics context, you draw into it, you're done. The system handles compositing that with other images over time (or rotating it in space, or whatever). So in the case of changing the hidden state, the initial-state fully composited image is blended with the final-state composted image. Very fast on a GPU, and it doesn't really matter what change you made to the view.

<https://stackoverflow.com/questions/2233692/how-does-catransition-work>



timingFunction





参考：

<http://liuyanwei.jumppo.com/2015/10/30/iOS-Animation-UIViewAndCoreAnimation.html>

<https://github.com/AsTryE/IOS-Teach/blob/master/1.IOS%E6%9C%80%E5%85%A8%E5%8A%A8%E7%94%BB%E6%95%99%E7%A8%8B%EF%BC%88%E5%9F%BA%E7%A1%80%EF%BC%89.md>