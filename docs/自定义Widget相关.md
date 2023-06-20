
<img src="https://s2.loli.net/2023/05/19/GYxKpOFJubvd5aQ.png" />

### 自定义Widget概述
**Widget 三大类**  
1. 作为『 Widget Tree 』的叶节点，也是最小的 UI 表达单元，一般继承自 LeafRenderObjectWidget；  
2. 有一个子节点 ( Single Child )，一般继承自 SingleChildRenderObjectWidget；  
3. 有多个子节点 ( Multi Child )，一般继承自 MultiChildRenderObjectWidget。  

对于RenderBox系列来说，如果要自定义子类，根据自定义子类子节点模型的不同需要有不同的处理：

1. 自定义子类本身是『 Render Tree 』的叶子节点，一般直接继承自 **RenderBox**

2. 有一个子节点 (Single Child)，且子节点属于RenderBox系列：  
- 如果其自身的 size 完全 匹配 子节点的 size，则可以选择继承自RenderProxyBox(如：RenderOffstage)
- 如果其自身的 size 大于子节点的 size，则可以选择继承自RenderShiftedBox(如：RenderPadding)

3. 有一个子节点 (Single Child)，但子节点不属于RenderBox系列，自定义子类可以 with RenderObjectWithChildMixin，其提供了管理一个子节点的模型 

4. 有多个子节点 (Multi Child)，自定义子类可以 with ContainerRenderObjectMixin、RenderBoxContainerDefaultsMixin，前者提供了管理多个子节点的模型，后者提供了基于ContainerRenderObjectMixin的一些默认实现

自定义 Widget 就是重写 createRenderObject 方法。

### 一般开发步骤
1. 根据需要继承对应的 ObjectWidget 并重写 createRenderObject 方法
```dart
class ScoreStar extends LeafRenderObjectWidget {
  final Color backgroundColor;
  final Color foregroundColor;
  final double score;

  ScoreStar(this.backgroundColor, this.foregroundColor, this.score, {super.key});

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderScoreStar(backgroundColor, foregroundColor, score);
  }
}
```

2. 继承 RenderObject , 一般会继承 RenderBox 作为 createRenderObject 方法的返回，同时传参进去
```dart
class RenderScoreStar extends RenderBox {}
```

3. 根据参数编写 get set 方法，并调用 markNeedsPaint
```dart
class RenderScoreStar extends RenderBox {
  Color _backgroundColor;
  //...
  RenderScoreStar(this._backgroundColor, this._foregroundColor, this._score);

  Color get backgroundColor => _backgroundColor;
  set backgroundColor(Color color) {
    _backgroundColor = color;
    markNeedsLayout();
  }
  //...
}
```
如果 不需要引起 layout 变化，则调用 markNeedsPaint，会引起 re-paint ，否则调用 markNeedsLayout

4. 重写 updateRenderObject 方法更新参数，其中参数 renderObject 改成自定义 RenderBox 的类型
```dart
  @override
  void updateRenderObject(BuildContext context, covariant RenderScoreStar renderObject) {
    super.updateRenderObject(context, renderObject);
    renderObject
      ..backgroundColor = backgroundColor
      ..foregroundColor = foregroundColor
      ..score = score;
  }
```

### 方法介绍

**sizedByParent**  
```dart
@override
  bool get sizedByParent => super.sizedByParent;
```

如果 RenderBox 的 size 完全由约束决定，则不需要重写（默认false），  
如果返回 true，则需求重写 **performResize** 方法来计算 size。  

但一般不会直接重写 performResize，而是重写 **computeDryLayout** 方法。  
因为 RenderBox 的 performResize 方法会调用 computeDryLayout ，并将返回结果作为当前组件的大小。
按照Flutter 框架约定，我们应该重写computeDryLayout 方法而不是 performResize 方法

若 sizedByParent 设为 false，则需要重写 performLayout，并在该方法中完成 size 的计算


**performLayout**  

若重写了performLayout方法，则进而需要重写以下四个方法：  
1. computeMaxIntrinsicWidth  用于计算一个**最小宽度**，在最终 size.width 超过该宽度时，也不会减少 size.height
2. computeMinIntrinsicWidth  排版需要的最小宽度，若小于这个宽度内容就会被裁剪；
3. computeMinIntrinsicHeight  //差不多
4. computeMaxIntrinsicHeight  //差不多

在一些特殊 RenderObject 排版时才会用到这些方法。其他根据 constraints 简单计算一下就行。

```dart
  @override
  double computeMaxIntrinsicWidth(double height) {
    return constraints.biggest.width;
  }
```

**hitTestSelf**  

如果需要响应用户事件，则需要重写 hitTestSelf 方法，并返回 true。
```dart
@override
bool hitTestSelf(Offset position) {
  return true;
}
```

### MultiChildRenderObjectWidget
ParentData 是在 layout 时使用的辅助定位信息。  
对于含有子节点的 RenderObject，一般都需要自定义自己的 ParentData 子类，用于辅助 layout。
```dart
class RichScoreParentData extends ContainerBoxParentData<RenderBox> {
  double scoreTextWidth;
}
```
继承 ContainerBoxParentData ，里面添加自己需要的字段.

实现 MultiChildRenderObjectWidget 时一般需要 with ContainerRenderObjectMixin 和 RenderBoxContainerDefaultsMixin
```dart
class RenderRichScore extends RenderBox with ContainerRenderObjectMixin<RenderBox, RichScoreParentData>,
    RenderBoxContainerDefaultsMixin<RenderBox, RichScoreParentData>,
    DebugOverflowIndicatorMixin {
  
}
```

MultiChildRenderObjectWidget 是有多个节点的，一般参数会有个 List<Widget> children，即 List<RenderBox> children  
在构造函数中，要调用 addAll 添加 children
```dart
  RenderRichScore({
    List<RenderBox> children,
  }) {
    addAll(children);
  }
```

之前自定义了 ContainerBoxParentData ，这时候一般需要重写 setupParentData 方法，为子节点设置 ParentData；
```dart
 @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! RichScoreParentData) {
      child.parentData = RichScoreParentData();
    }
  }
```

需要重写 computeDistanceToActualBaseline 计算 Baseline
```dart
  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    return defaultComputeDistanceToFirstActualBaseline(baseline);
  }
```