1. 通过静态扩展类DOTweenModuleUI来扩展UI相关组件，把UI组件接入DOTween。接入的UI组件

   ![](https://i.loli.net/2021/01/15/IErSOCkXo71btAa.png)

   函数的基本样式都是传入要tween的target，最终要达到的值endvalue，以及持续时间，然后返回TweenerCore或者Sequence对象。针对不同的UI组件会具备对应的Tween操作，如下:

   CanvasGroup: DOFade.

   Graphic: DOColor, DOFade.

   Image: DOColor, DOFade, DOFillAmount , DOGradientColor.

   LayoutElement: DOFlexibleSize, DOMinSize, DOPreferredSize.

   Outline: DOColor, DOFade, DOScale.

   RectTransform: DOAnchorPos(XY) , DOAnchorPos3D(XYZ) , DOAnchorMax(Min), DOPivot(XY), DOSizeDelta, DOPunchAnchorPos, DOShakeAnchorPos.

   ScrollRect: DONormalizedPos, DOHorizontalNormalizedPos, DOVerticalNormalizedPos.

   Slider: DOValue.

   Text: DOClolor, DOFade, DOText.

   Blendables: Graphic\Image\Text DOBlendableColor.

   是组件属性的子集，通过这些扩展方法可以很方便的对属性进行插值和赋值。

2. DOTween提供给外部访问的接口，内部会构造一个TweenerCore对象并返回出来。API都需要提供属性的setter和getter以便访问，通过DOGetter和DOSetter两个委托约束。通过这两个委托在tween时来访问属性并将插值结果运用到属性。API内部通过ApplyTo来构造TweenerCore。通过传入getter，setter，endvalue，duration和用户自己实现的自定义属性插值器（只需要继承ABSTweenPlugin抽象类即可）来构造，这个时候会init Tween的setting以及在scene中创建一个DOTween的代理gameobject并加上DOTweenComponent，最后是通过TweenManager获取一个类型匹配的TweenCore，然后把getter、setter等属性SetUp上去并进行一些检查，设置成功即可返回，否者返回空，并Despawn把对象返回给TweenManager，根据tween是否isRecyclable决定直接销毁或是放回pool。

3. TweenManager，一个Tween的管理类，用stack管理Sequence和用list管理Tweener，并记录分别记录了其中tweener和sequence的总数量，pool数量，active数量，还对不同更新模式的tween进行了计数，Default、Fixed、Late、Manual等。

   主要功能，GetTweener获取一个Tweener，内部会根据pool里面数量是否足够确定是新建还是从pool里面复用，同时根据参数进行匹配，获得一个Tweener，然后AddActiveTween加入到active列表里面。另一个方法GetSequence类似，是获取一个Sequence。Despawn返还一个Tween，DespawnAll返还所有Tween。

4. TweenrCore是一个泛型类，继承Tweener，需要getter和setter的属性类型，

5. Tweener和Sequence都继承Tween，Tween继承ANSSequentiable。
