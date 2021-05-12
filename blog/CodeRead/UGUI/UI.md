> <center><font face="黑体" size=32>UI</font></center>

[TOC]

UI组件部分代码比较多，但是游戏中UI常见的性能问题，如rebuild、relayout也发生在这，所以是比较重要的一部分。内容将分为以下几部分，core component，layout、cull、rebuild、relayout。

##### Core Component

- Selectable，如字面意思selectable提供了基本的交互功能。

  ![selectable](UI.assets/selectable.PNG)

  Inspector面板上有上面类容的组件都是继承自selectable，在此基础上进行自定义的功能扩展。Selectable实现了**IMoveHandler, IPointerDownHandler,  IPointerUpHandler, IPointerEnterHandler,  IPointerExitHandler, ISelectHandler,  IDeselectHandler**几个接口。所以具备对enter/exit，down/up，select/deselect以及navigation move事件的基本响应。

  move事件，实现IMoveHandler中的OnMove方法，以支持我们之前提到的navigation事件，也就是对上下左右、wasd键进行响应。

  ```c#
  public virtual void OnMove(AxisEventData eventData)
  {
      switch (eventData.moveDir)
      {
          case MoveDirection.Right:
              Navigate(eventData, FindSelectableOnRight());
              break;
              ...
      }
  }
  
  void Navigate(AxisEventData eventData, Selectable sel)
  {
      if (sel != null && sel.IsActive())
          eventData.selectedObject = sel.gameObject;
  }
  ```

  根据dir选择对应的selectable，然后把它设置为当前选中的selectedObject。选择对应的方向的selectable方式要视navigation的模式而定。如选择右边的selectable。

  ```c#
  public virtual Selectable FindSelectableOnRight()
  {
      if (m_Navigation.mode == Navigation.Mode.Explicit)
      {
          return m_Navigation.selectOnRight;
      }
      if ((m_Navigation.mode & Navigation.Mode.Horizontal) != 0)
      {
          return FindSelectable(transform.rotation * Vector3.right);
      }
      return null;
  }
  ```

  如果被精确指定也就是Explicit模式，就会取出设置的selectable。如Dropdown里面的每个Toggle都会设置为该模式，并在下拉框显示的时候进行初始化设置每个方向的selectable，如Right和Down方向设置为列表中的下一个selectable，而Left和Up方向设置为列表中的上一个selectable。

  ![togglenavigation](UI.assets/togglenavigation.PNG)

  而scrollbar的navigation就是Automatic需要调用FindSelectable来查找合适的selectable，这儿需要所有的selectable信息，所以selectable中维护了一个当前所有激活的selectable列表。

  ![scrollbarnavigation](UI.assets/scrollbarnavigation.PNG)

  ```c#
  private static Selectable[] s_Selectables = new Selectable[10];
  ```

  选取的规则是，两个selectable的方向向量要和查询方向一致，同时在方向上的投影应该尽可能的长，最后选取的是score最大的selectable。

  ```c#
  float dot = Vector3.Dot(dir, myVector);
  if (dot <= 0)
      continue;
  float score = dot / myVector.sqrMagnitude;
  ```

  PointerDown、PointerUp、PointerEnter、PointerExit、Select、Deselect事件，都是设置selectable的状态，并进行相应表现。维护了一个当前状态。

  ```c#
  protected SelectionState currentSelectionState
  {
      get
      {
          if (!IsInteractable())
              return SelectionState.Disabled;
          if (isPointerDown)
              return SelectionState.Pressed;
          if (hasSelection)
              return SelectionState.Selected;
          if (isPointerInside)
              return SelectionState.Highlighted;
          return SelectionState.Normal;
      }
  }
  ```

  以及相应的转换函数。

  ```c#
  switch (state)
  {
      case SelectionState.Normal:
          tintColor = m_Colors.normalColor;
          transitionSprite = null;
          triggerName = m_AnimationTriggers.normalTrigger;
          break;
      case SelectionState.Highlighted:
          tintColor = m_Colors.highlightedColor;
          transitionSprite = m_SpriteState.highlightedSprite;
          triggerName = m_AnimationTriggers.highlightedTrigger;
          break;
      case SelectionState.Pressed:
          tintColor = m_Colors.pressedColor;
          transitionSprite = m_SpriteState.pressedSprite;
          triggerName = m_AnimationTriggers.pressedTrigger;
          break;
      case SelectionState.Selected:
          tintColor = m_Colors.selectedColor;
          transitionSprite = m_SpriteState.selectedSprite;
          triggerName = m_AnimationTriggers.selectedTrigger;
          break;
      case SelectionState.Disabled:
          tintColor = m_Colors.disabledColor;
          transitionSprite = m_SpriteState.disabledSprite;
          triggerName = m_AnimationTriggers.disabledTrigger;
          break;
      default:
          tintColor = Color.black;
          transitionSprite = null;
          triggerName = string.Empty;
          break;
  }
  
  switch (m_Transition)
  {
      case Transition.ColorTint:
          StartColorTween(tintColor * m_Colors.colorMultiplier, instant);
          break;
      case Transition.SpriteSwap:
          DoSpriteSwap(transitionSprite);
          break;
      case Transition.Animation:
          TriggerAnimation(triggerName);
          break;
  }
  ```

  selectable支持3种转换方式，基于color、sprite、animation。color和sprite都需要基于selectable gameobject上面的Graphic或Image来做切换，sprite是直接切，color有一个切换的过程，而animation需要有Animator组件并设置上面对应的trigger。color的切换支持一个过渡时间，通过graphic中的**CrossFadeColor**进行。

  ```c#
  var colorTween = new ColorTween {duration = duration, startColor = canvasRenderer.GetColor(), targetColor = targetColor};
  colorTween.AddOnChangedCallback(canvasRenderer.SetColor);
  colorTween.ignoreTimeScale = ignoreTimeScale;
  colorTween.tweenMode = mode;
  m_ColorTweenRunner.StartTween(colorTween);
  ```

  构造了一个ColorTween结构，然后用ColorTweenRunner来run。TweenRunner是基于当前graphic构建的run tween动画的类，内部用当前monobehaviour通过协程的方式来驱动动画。

  ```c#
  internal class TweenRunner<T> where T : struct, ITweenValue{}
  internal struct ColorTween : ITweenValue{}
  internal struct FloatTween : ITweenValue{}
  ```

  继承自selectable的有以下组件：InputField、Toggle、Button、Dropdown、Slider、Scrollbar。

- Graphic，之前提到过graphic提供给CanvasRenderer所需的mesh数据，Graphic+CanvasRenderer类似，Mesh Filter+Mesh Renderer的关系。遗憾是的没有**Canvas**和**CanvasRenderer**的源码。graphic不能直接添加到gameobject因为这是个抽象类，实现它的类MaskableGraphic也是个抽象类，但是**Image、Text、RawImage**都继承自MaskableGraphic，所以他们都具备Graphic提供的Material和Raycast功能，以及MaskableGraphic提供的可以被cull掉。

  其中包含公用的defaultMaterial和default的whiteTexture，以及用来辅助mesh构建的workerMesh。

  ```c#
  public virtual Material material
  {
      get
      {
          return (m_Material != null) ? m_Material : defaultMaterial;
      }
  }
  
  protected static Mesh workerMesh
  {
      get
      {
          if (s_Mesh == null)
          {
              s_Mesh = new Mesh();
              s_Mesh.name = "Shared UI Mesh";
              s_Mesh.hideFlags = HideFlags.HideAndDontSave;
          }
          return s_Mesh;
      }
  }
  ```

  OnEnable中需要把graphic注册到GraphicRegistry中以供raycast使用，同时会SetAllDirty。

  ```c#
  protected override void OnEnable()
  {
      GraphicRegistry.RegisterGraphicForCanvas(canvas, this);
      SetAllDirty();
  }
  ```


##### Layout

- LayoutRebuilder，内部维护了一个自己的对象池，每一个rebuilder都通过m_ToRebuild来维护该rebuilder当前绑定的transform，也就是rebuild的对象。同时UGUI的对象池实现有一个值得借鉴的地方，就是每从对象池中get一个对象，或换给对象池的时候，都会有一个回调可以绑定，这样可以确保用户正确的设置对象的状态，即对象池不关注对象的状态，这都交给用户来考虑。当然一般的建议是，对象池里面的对象是一个比较纯粹的，即用户需要保证在还给对象池的时候，清除自己的修改记录，这样引用到的资源也可以清理掉。

  ```c#
  private RectTransform m_ToRebuild;
  ...
  static ObjectPool<LayoutRebuilder> s_Rebuilders = new ObjectPool<LayoutRebuilder>(null, x => x.Clear());
  ```

  在RectTransform调用MarkLayoutForRebuild的时候就会对rect进行对应的rebuild检查。如Graphic OnEable或者修改Parent的时候都会调用MarkAllDirty，从而调用MarkLayoutForRebuild。该方法会向上遍历，找到连续的最上面一个实现ILayoutGroup(继承ILayoutController)的节点，并加入到CanvasUpdateRegistry中的LayoutRebuild队列中。

  ```c#
  var rebuilder = s_Rebuilders.Get();
  	rebuilder.Initialize(controller);
  if (!CanvasUpdateRegistry.TryRegisterCanvasElementForLayoutRebuild(rebuilder))
      s_Rebuilders.Release(rebuilder);
  ```

  ```c#
  private bool m_PerformingLayoutUpdate;
  private bool m_PerformingGraphicUpdate;
  
  private readonly IndexedSet<ICanvasElement> m_LayoutRebuildQueue = new IndexedSet<ICanvasElement>();
  private readonly IndexedSet<ICanvasElement> m_GraphicRebuildQueue = new IndexedSet<ICanvasElement>();
  ```

  同时ContentSizeFitter，LayoutElement，LayoutGroup，ScrollRect都会相应的触发MarkLayoutForRebuild。需要注意的是如果当前正在relayout，此时再加入relayout的transform会出现一些异常的情况，第一中加入不进去，报错。但是在比较新的版本已经被注释掉了。

  ```c#
  private bool InternalRegisterCanvasElementForLayoutRebuild(ICanvasElement element)
  {
      if (m_LayoutRebuildQueue.Contains(element))
          return false;
  
      /* TODO: this likely should be here but causes the error to show just resizing the game view (case 739376)
              if (m_PerformingLayoutUpdate)
              {
                  Debug.LogError(string.Format("Trying to add {0} for layout rebuild while we are already inside a layout rebuild loop. This is not supported.", element));
                  return false;
              }*/
  
      return m_LayoutRebuildQueue.AddUnique(element);
  }
  ```

  第二种，对于LayoutGroup会把它延迟到下一帧再MarkLayoutForRebuild。

  ```c#
  IEnumerator DelayedSetDirty(RectTransform rectTransform)
  {
      yield return null;
      LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
  }
  ```

- HorizontalLayoutGroup和VerticalLayoutGroup都继承自HorizontalOrVerticalLayoutGroup，  HorizontalOrVerticalLayoutGroup和GridLayoutGroup都继承了LayoutGroup，最后LayoutGroup实现了 ILayoutElement、ILayoutGroup。具体的relayout流程，首先把所有的layout根据parent数量排序，即更浅层节点先rebuild。具体的rebuild逻辑再LayoutRebuilder里面。

  ```c#
  public void Rebuild(CanvasUpdate executing)
  {
      switch (executing)
      {
          case CanvasUpdate.Layout:
              PerformLayoutCalculation(m_ToRebuild, e => (e as ILayoutElement).CalculateLayoutInputHorizontal());
              PerformLayoutControl(m_ToRebuild, e => (e as ILayoutController).SetLayoutHorizontal());
              PerformLayoutCalculation(m_ToRebuild, e => (e as ILayoutElement).CalculateLayoutInputVertical());
              PerformLayoutControl(m_ToRebuild, e => (e as ILayoutController).SetLayoutVertical());
              break;
      }
  }
  ```

  先进行horizontal方向的relayout，再进行vertical方向的relayout，具体的实现在Horizontal、Vertical和Grid三个布局器里面。主要就是计算对应轴方向上的min，preferred、flexible三个值，当然布局器的这几个值是通过child算出来的，所以在真正的计算之前需要拿到布局器下的child。

  ```c#
   for (int i = 0; i < rectTransform.childCount; i++)
   {
       var rect = rectTransform.GetChild(i) as RectTransform;
       if (rect == null || !rect.gameObject.activeInHierarchy)
           continue;
  
       rect.GetComponents(typeof(ILayoutIgnorer), toIgnoreList);
  
       if (toIgnoreList.Count == 0)
       {
           m_RectChildren.Add(rect);
           continue;
       }
  
       for (int j = 0; j < toIgnoreList.Count; j++)
       {
           var ignorer = (ILayoutIgnorer)toIgnoreList[j];
           if (!ignorer.ignoreLayout)
           {
               m_RectChildren.Add(rect);
               break;
           }
       }
   }
  ```

  可以看到ILayoutIgnorer就是在这儿用来过滤child 的，表示在计算的时候不考虑被ILayoutIgnorer标记的child。

  真正的计算通过child的min、preferred、flexible来指导，通过LayoutUtility中的函数获取具体的值。

  ```c#
  public static float GetMinSize(RectTransform rect, int axis)
  {
      if (axis == 0)
          return GetMinWidth(rect);
      return GetMinHeight(rect);
  }
  
  public static float GetMinHeight(RectTransform rect)
  {
      return GetLayoutProperty(rect, e => e.minHeight, 0);
  }
  ```

  具体的GetLayoutProperty实现。

  ```c#
  rect.GetComponents(typeof(ILayoutElement), components);
  for (int i = 0; i < components.Count; i++)
  {
      var layoutComp = components[i] as ILayoutElement;
      if (layoutComp is Behaviour && !((Behaviour)layoutComp).isActiveAndEnabled)
          continue;
  
      int priority = layoutComp.layoutPriority;
      // If this layout components has lower priority than a previously used, ignore it.
      if (priority < maxPriority)
          continue;
      float prop = property(layoutComp);
      // If this layout property is set to a negative value, it means it should be ignored.
      if (prop < 0)
          continue;
  
      // If this layout component has higher priority than all previous ones,
      // overwrite with this one's value.
      if (priority > maxPriority)
      {
          min = prop;
          maxPriority = priority;
          source = layoutComp;
      }
      // If the layout component has the same priority as a previously used,
      // use the largest of the values with the same priority.
      else if (prop > min)
      {
          min = prop;
          source = layoutComp;
      }
  }
  ```

  可以看到一个transform上可以有恨过给ILayoutElement来指导具体的值的计算，最后是通过优先级来确定最终的值的。像LayoutElement实现了ILayoutElement，并把对应的值暴露出来给用户设置。而text实现ILayoutElement实现了自己的preferredWidth，这样便可以通过具体的font和text信息算出刚好包含text的组将宽度。

  ```c#
  public virtual float preferredWidth
  {
      get
      {
          var settings = GetGenerationSettings(Vector2.zero);
          return cachedTextGeneratorForLayout.GetPreferredWidth(m_Text, settings) / pixelsPerUnit;
      }
  }
  ```

  OK，这样我们可以算出layout需要的height或width，然后就是设置layout的宽度并设置child的位置。注意ILayoutSelfController继承了ILayoutController，所以实现了ILayoutSelfController的ContentSizeFitter也可以修改到自己的size。

##### Cull

- 通过mask使用stencil来cull，前面提到MaskableGraphic是支持stencil mask的，实现了IMaskable接口，IMaskable中有RecalculateMasking方法。

  ```c#
  public virtual void RecalculateMasking()
  {
      StencilMaterial.Remove(m_MaskMaterial);
      m_MaskMaterial = null;
      m_ShouldRecalculateStencil = true;
      SetMaterialDirty();
  }
  ```

  MaskableGraphic中会维护一个mask mat，mask mat通过StencilMaterial来统一管理。主要记录了stencil条件和baseMat，使得具有相同stencil条件和mat的组件可以共享mat，避免创建太多的重复的stancil mat，用引用计数维护这些stencil mat的声明周期。

  ```c#
  private class MatEntry
  {
      public Material baseMat;
      public Material customMat;
      public int count;
  
      public int stencilId;
      public StencilOp operation = StencilOp.Keep;
      public CompareFunction compareFunction = CompareFunction.Always;
      public int readMask;
      public int writeMask;
      public bool useAlphaClip;
      public ColorWriteMask colorMask;
  }
  
  private static List<MatEntry> m_List = new List<MatEntry>();
  ```

  Remove的时候。

  ```c#
  public static void Remove(Material customMat)
  {
      if (customMat == null)
          return;
  
      for (int i = 0; i < m_List.Count; ++i)
      {
          MatEntry ent = m_List[i];
  
          if (ent.customMat != customMat)
              continue;
  
          if (--ent.count == 0)
          {
              Misc.DestroyImmediate(ent.customMat);
              ent.baseMat = null;
              m_List.RemoveAt(i);
          }
          return;
      }
  }
  ```

  通过前面的代码可以看到，触发RecalculateMasking的时候会把mat设置为dirty，然后会放入Graphic Rebuild队列中。

  ```c#
  public virtual void SetMaterialDirty()
  {
      ...
      CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
  }
  ```

  graphic rebuild的时候会调用对应的Graphic Rebuild。

  ```c#
  public virtual void Rebuild(CanvasUpdate update)
  {
  	...
      switch (update)
      {
          case CanvasUpdate.PreRender:
              if (m_VertsDirty)
              {
                  UpdateGeometry();
                  m_VertsDirty = false;
              }
              if (m_MaterialDirty)
              {
                  UpdateMaterial();
                  m_MaterialDirty = false;
              }
              break;
      }
  }
  ```

  里面会根据自己的dirty类型进行具体的rebuild操作，此处是mat dirty，所以会进入UpdateMaterial。

  ```c#
  protected virtual void UpdateMaterial()
  {
      if (!IsActive())
          return;
  
      canvasRenderer.materialCount = 1;
      canvasRenderer.SetMaterial(materialForRendering, 0);
      canvasRenderer.SetTexture(mainTexture);
  }
  ```

  注意上面的materialForRendering就是最后运用的材质，他的具体获取。

  ```c#
  public virtual Material materialForRendering
  {
      get
      {
          var components = ListPool<Component>.Get();
          GetComponents(typeof(IMaterialModifier), components);
  
          var currentMat = material;
          for (var i = 0; i < components.Count; i++)
              currentMat = (components[i] as IMaterialModifier).GetModifiedMaterial(currentMat);
          ListPool<Component>.Release(components);
          return currentMat;
      }
  }
  ```

  从上面代码可以看出，会用实现了IMaterialModifier的组件来对材质进行修改。MaskableGraphic也实现了该接口。其中GetModifiedMaterial的具体实现为。

  ```c#
  public virtual Material GetModifiedMaterial(Material baseMaterial)
  {
      var toUse = baseMaterial;
  
      if (m_ShouldRecalculateStencil)
      {
          var rootCanvas = MaskUtilities.FindRootSortOverrideCanvas(transform);
          m_StencilValue = maskable ? MaskUtilities.GetStencilDepth(transform, rootCanvas) : 0;
          m_ShouldRecalculateStencil = false;
      }
  
      // if we have a enabled Mask component then it will
      // generate the mask material. This is an optimisation
      // it adds some coupling between components though :(
      Mask maskComponent = GetComponent<Mask>();
      if (m_StencilValue > 0 && (maskComponent == null || !maskComponent.IsActive()))
      {
          var maskMat = StencilMaterial.Add(toUse, (1 << m_StencilValue) - 1, StencilOp.Keep, CompareFunction.Equal, ColorWriteMask.All, (1 << m_StencilValue) - 1, 0);
          StencilMaterial.Remove(m_MaskMaterial);
          m_MaskMaterial = maskMat;
          toUse = m_MaskMaterial;
      }
      return toUse;
  }
  ```

  通过MaskUtilities找到第一个override sort order的或者是最上层的canvas，然后通过GetStencilDepth算出stencil depth。

  ```c#
  public static int GetStencilDepth(Transform transform, Transform stopAfter)
  {
      var depth = 0;
      if (transform == stopAfter)
          return depth;
      var components = ListPool<Mask>.Get();
      while (t != null)
      {
          t.GetComponents<Mask>(components);
          for (var i = 0; i < components.Count; ++i)
          {
              if (components[i] != null && components[i].MaskEnabled() && components[i].graphic.IsActive())
              {
                  ++depth;
                  break;
              }
          }
          if (t == stopAfter)
              break;
          t = t.parent;
      }
      ListPool<Mask>.Release(components);
      return depth;
  }
  ```

  也就是统计到stop节点的路径上有多少个激活的mask。然后根据stencil value从StencilMaterial获取一个对应的stencil mat。当然，没有激活的mask就用basemat就可以了 。

  ```c#
  public static Material Add(Material baseMat, int stencilID, StencilOp operation, CompareFunction compareFunction, ColorWriteMask colorWriteMask, int readMask, int writeMask)
  {
      if ((stencilID <= 0 && colorWriteMask == ColorWriteMask.All) || baseMat == null)
          return baseMat;
      	...
          for (int i = 0; i < m_List.Count; ++i)
          {
              MatEntry ent = m_List[i];
  
              if (ent.baseMat == baseMat
                  && ent.stencilId == stencilID
                  && ent.operation == operation
                  && ent.compareFunction == compareFunction
                  && ent.readMask == readMask
                  && ent.writeMask == writeMask
                  && ent.colorMask == colorWriteMask)
              {
                  ++ent.count;
                  return ent.customMat;
              }
          }
  
      var newEnt = new MatEntry();
      newEnt.count = 1;
      newEnt.baseMat = baseMat;
      newEnt.customMat = new Material(baseMat);
      newEnt.customMat.hideFlags = HideFlags.HideAndDontSave;
      newEnt.stencilId = stencilID;
      newEnt.operation = operation;
      newEnt.compareFunction = compareFunction;
      newEnt.readMask = readMask;
      newEnt.writeMask = writeMask;
      newEnt.colorMask = colorWriteMask;
      newEnt.useAlphaClip = operation != StencilOp.Keep && writeMask > 0;
      ...
      newEnt.customMat.SetInt("_Stencil", stencilID);
      newEnt.customMat.SetInt("_StencilOp", (int)operation);
      newEnt.customMat.SetInt("_StencilComp", (int)compareFunction);
      newEnt.customMat.SetInt("_StencilReadMask", readMask);
      newEnt.customMat.SetInt("_StencilWriteMask", writeMask);
      newEnt.customMat.SetInt("_ColorMask", (int)colorWriteMask);
  
      m_List.Add(newEnt);
      return newEnt.customMat;
  }
  ```

  stencil mat是clone了一个basemat，然后修改上面相关的stencil设置，所以需要basemat也有相应的stencil property支持。主要就是设置stencilID，readmask和writemask。注意stencil id和readmask都是，**(1 << m_StencilValue) - 1**，compare function是equal，当然writemask是0。

  OK，我们给需要mask的组件换上了有stencil test的材质，那么谁来写入stencil呢？理所当然，是mask。mask enable的时候。

  ```c#
  protected override void OnEnable()
  {
      base.OnEnable();
      if (graphic != null)
      {
          graphic.canvasRenderer.hasPopInstruction = true;
          graphic.SetMaterialDirty();
      }
      MaskUtilities.NotifyStencilStateChanged(this);
  }
  ```

  注意这儿为CanvasRenderer设置了**hasPopInstruction**标志，后面会说明其用处。同样mask实现了GetModifiedMaterial以再rebuild的时候设置自己的材质。

  ```c#
  if (!MaskEnabled())
  return baseMaterial;
  
  var rootSortCanvas = MaskUtilities.FindRootSortOverrideCanvas(transform);
  var stencilDepth = MaskUtilities.GetStencilDepth(transform, rootSortCanvas);
  if (stencilDepth >= 8)
  {
  Debug.LogError("Attempting to use a stencil mask with depth > 8", gameObject);
  return baseMaterial;
  }
  
  int desiredStencilBit = 1 << stencilDepth;
  ```

  首先，计算一下自己占用stencil buffer中的第几位，显然，做多使用8位，如我们经常分配的depth_stencil buffer是depth占用24bit，stencil占用8bit。

  ```c#
  if (desiredStencilBit == 1)
  {
      var maskMaterial = StencilMaterial.Add(baseMaterial, 1, StencilOp.Replace, CompareFunction.Always, m_ShowMaskGraphic ? ColorWriteMask.All : 0);
      StencilMaterial.Remove(m_MaskMaterial);
      m_MaskMaterial = maskMaterial;
  
      var unmaskMaterial = StencilMaterial.Add(baseMaterial, 1, StencilOp.Zero, CompareFunction.Always, 0);
      StencilMaterial.Remove(m_UnmaskMaterial);
      m_UnmaskMaterial = unmaskMaterial;
      graphic.canvasRenderer.popMaterialCount = 1;
      graphic.canvasRenderer.SetPopMaterial(m_UnmaskMaterial, 0);
  
      return m_MaskMaterial;
  }
  ```

  每个mask会创建两个材质，一个是开始渲染自己时用来写入stencil的，一个是在结束自己渲染的时候用来clear stencil的，即m_MaskMaterial和m_UnmaskMaterial，同时设置popMaterialCount表明在渲染完成自己和下面的子节点后，有额外的一步render，并把对应的unmask材质赋值。

  对于最上层的mask，直接写入最后一位，同时clear的时候直接清零即可。而对于更下层的mask需要一点额外的处理。

  ```c#
  var maskMaterial2 = StencilMaterial.Add(baseMaterial, desiredStencilBit | (desiredStencilBit - 1), StencilOp.Replace, CompareFunction.Equal, m_ShowMaskGraphic ? ColorWriteMask.All : 0, desiredStencilBit - 1, desiredStencilBit | (desiredStencilBit - 1));
  StencilMaterial.Remove(m_MaskMaterial);
  m_MaskMaterial = maskMaterial2;
  
  graphic.canvasRenderer.hasPopInstruction = true;
  var unmaskMaterial2 = StencilMaterial.Add(baseMaterial, desiredStencilBit - 1, StencilOp.Replace, CompareFunction.Equal, 0, desiredStencilBit - 1, desiredStencilBit | (desiredStencilBit - 1));
  ```

  也就是，读取的时候比较前n-1位，写的时候写第n位。

  当mask enable和disable的时候会通过 MaskUtilities的NotifyStencilStateChanged通知其所有的IMaskable子节点调用RecalculateMasking，像我们前面看到的MaskableGraphic会更新自己的材质。

- 通过rect用RectMask2D来cull，另一种裁剪的方式是通过实现IClipper接口，在方法PerformClipping里面实现cull。RectMask2D就是实现了该接口。

  ```c#
  private List<RectMask2D> m_Clippers = new List<RectMask2D>();
  if (m_ShouldRecalculateClipRects)
  {
      MaskUtilities.GetRectMasksForClip(this, m_Clippers);
      m_ShouldRecalculateClipRects = false;
  }
  ```

  首先判断是不是要重新计算clippers，也就是找出从自己到根节点上所有的RectMask2D，然后计算clipRect，如果相比之前的clipRect有变化，就把clipRect同步给所有的m_ClipTargets。

  ```c#
  private HashSet<IClippable> m_ClipTargets = new HashSet<IClippable>();
  foreach (IClippable clipTarget in m_ClipTargets)
  {
      if (clipRectChanged || forceClip)
          clipTarget.SetClipRect(clipRect, validRect);
      var maskable = clipTarget as MaskableGraphic;
      if (maskable != null && !maskable.canvasRenderer.hasMoved && !clipRectChanged)
          continue;
      clipTarget.Cull( maskIsCulled ? Rect.zero : clipRect, maskIsCulled ? false : validRect);
  }
  ```

  m_ClipTargets是IClippable列表，MaskableGraphic实现了该接口。

  ```c#
  public virtual void SetClipRect(Rect clipRect, bool validRect)
  {
      if (validRect)
          canvasRenderer.EnableRectClipping(clipRect);
      else
          canvasRenderer.DisableRectClipping();
  }
  ```

  Cull是判断是不是要完整的cull掉节点。

  ```c#
   public virtual void Cull(Rect clipRect, bool validRect)
   {
       var cull = !validRect || !clipRect.Overlaps(rootCanvasRect, true);
       UpdateCull(cull);
   }
  
  private void UpdateCull(bool cull)
  {
      if (canvasRenderer.cull != cull)
          canvasRenderer.cull = cull;
  }
  ```

  如果rect和canvas的rect没有交集，就可以直接cull掉了。

  所有的clpper都被ClipperRegistry管理，在RectMask2D enable和disable会对应的往里面注册和反注册，在注册的时候还会notify下面的IClippable RecalculateClipping。

  ```c#
  protected override void OnEnable()
  {
      m_ShouldRecalculateClipRects = true;
      ClipperRegistry.Register(this);
      MaskUtilities.Notify2DMaskStateChanged(this);
  }
  
  protected override void OnDisable()
  {
      m_ClipTargets.Clear();
      m_Clippers.Clear();
      ClipperRegistry.Unregister(this);
      MaskUtilities.Notify2DMaskStateChanged(this);
  }
  ```

  ClipperRegistry是个单例，维护所有的clipper并驱动cull，当然ClipperRegistry的cull最终是被CanvasUpdateRegistry的PerformUpdate驱动。

  ```c#
  readonly IndexedSet<IClipper> m_Clippers = new IndexedSet<IClipper>();
  
  public void Cull()
  {
      for (var i = 0; i < m_Clippers.Count; ++i)
          m_Clippers[i].PerformClipping();
  }
  
  ClipperRegistry.instance.Cull();
  ```

  最后就是RectMask2D中的ClipTargets的维护，

  ```c#
  private HashSet<IClippable> m_ClipTargets = new HashSet<IClippable>();
  public void AddClippable(IClippable clippable)
  {
      if (clippable == null)
          return;
      m_ShouldRecalculateClipRects = true;
      if (!m_ClipTargets.Contains(clippable))
          m_ClipTargets.Add(clippable);
      m_ForceClip = true;
  }
  
  public void RemoveClippable(IClippable clippable)
  {
      if (clippable == null)
          return;
      m_ShouldRecalculateClipRects = true;
      clippable.SetClipRect(new Rect(), false);
      m_ClipTargets.Remove(clippable);
      m_ForceClip = true;
  }
  ```

  AddClippable、RemoveClippable会在MaskableGraphic中的UpdateClipParent中调用，前面说到的RectMask2D enable的时候注册自己的时候会notify child IClippable，其中调用RecalculateClipping时就会调用UpdateClipParent把自己注册到对应的parent IClipper中。

  ```c#
  var newParent = (maskable && IsActive()) ? MaskUtilities.GetRectMaskForClippable(this) : null;
  if (m_ParentMask != null && (newParent != m_ParentMask || !newParent.IsActive()))
  {
      m_ParentMask.RemoveClippable(this);
      UpdateCull(false);
  }
  if (newParent != null && newParent.IsActive())
      newParent.AddClippable(this);
  m_ParentMask = newParent;
  ```

  可以看出，RectMask2D没有使用stencil并且也不会有额外的pop instruction，所以不会打断batch。

##### Rebuild

前面我们已经看到了Rebuildid一部分功能，用来UpdateMaterial，另一个功能时UpdateGeometry。material dirty和vertex dirty的时候都会把Graphic加入rebuild队列中，最后回调到Rebuild函数的时候，会根据dirty的类型进行具体的rebuild。当组件enable的时候，以及设置material的时候会mark material dirty。而mark vertex dirty发生在组件enable，设置graphic color(因为用的是vertex color)，以及Transform Dimension改变的时候。

但是需要注意的是，上面提到的只是graphic中会mark dirty的情况，实际继承graphic的组件会有更多的情况mark dirty。如image中，会在SetNativeSize，Type以及设置FillMethod的时候进行相应的rebuild。

```c#
public virtual void Rebuild(CanvasUpdate update)
{
    if (canvasRenderer.cull)
        return;

    switch (update)
    {
        case CanvasUpdate.PreRender:
            if (m_VertsDirty)
            {
                UpdateGeometry();
                m_VertsDirty = false;
            }
            if (m_MaterialDirty)
            {
                UpdateMaterial();
                m_MaterialDirty = false;
            }
            break;
    }
}
```

UpdateMaterial就是直接设置canvas renderer使用的mat。

```c#
protected virtual void UpdateGeometry()
{
    if (useLegacyMeshGeneration)
        DoLegacyMeshGeneration();
    else
        DoMeshGeneration();
}
```

有两种生成mesh的方式，遗留的和新的。主要是进行了一些warning的处理。

mesh生成方式主要分为以下步骤，首先填充s_VertexHelper其中是记录了vertex、uv等信息的list，然后查询object上实现了IMeshModifier的组件，实现了IMeshModifier的组件拥有进一步修改mesh的机会，最后把根据vertexhelper中的信息构造mesh，赋值给CanvasRenderer。注意其中用到的辅助结构VertexHelper和WorkMesh都是static。需要注意的是**由于VertexHelper中的数据是用的一堆list保存的所以一开始的时候会有resize的开销**，但是有一个小trick后面会提到。unity内部的BaseMeshEffect就实现了IMeshModifier。

OnPopulateMesh就是直接往VertexHelper中填充数据。

```c#
protected virtual void OnPopulateMesh(VertexHelper vh)
{
    var r = GetPixelAdjustedRect();
    var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);

    Color32 color32 = color;
    vh.Clear();
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(0f, 0f));
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(0f, 1f));
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(1f, 1f));
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(1f, 0f));

    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

显然这是一个普通的Quad，更复杂的mesh生成需要组件重写该方法。如Image中会根据type类型来生成对应的mesh。

```c#
if (activeSprite == null)
{
    base.OnPopulateMesh(toFill);
    return;
}

switch (type)
{
    case Type.Simple:
        if (!useSpriteMesh)
            GenerateSimpleSprite(toFill, m_PreserveAspect);
        else
            GenerateSprite(toFill, m_PreserveAspect);
        break;
    case Type.Sliced:
        GenerateSlicedSprite(toFill);
        break;
    case Type.Tiled:
        GenerateTiledSprite(toFill);
        break;
    case Type.Filled:
        GenerateFilledSprite(toFill, m_PreserveAspect);
        break;
}
```

##### Text

text单独拿出来讲，主要是因为相对比较复杂，设计到font、glyph，baseline、advance、linespace等信息。同时还会动态的更新字体贴图，生成字体mesh。

```c#
protected override void OnPopulateMesh(VertexHelper toFill)
{
    if (font == null)
        return;
    var settings = GetGenerationSettings(extents);
    cachedTextGenerator.PopulateWithErrors(text, settings, gameObject);
	...
    IList<UIVertex> verts = cachedTextGenerator.verts;
    float unitsPerPixel = 1 / pixelsPerUnit;
    int vertCount = verts.Count - 4;
	...
    
    if (roundingOffset != Vector2.zero)
    {
        for (int i = 0; i < vertCount; ++i)
        {
            int tempVertsIndex = i & 3;
            m_TempVerts[tempVertsIndex] = verts[i];
            m_TempVerts[tempVertsIndex].position *= unitsPerPixel;
            m_TempVerts[tempVertsIndex].position.x += roundingOffset.x;
            m_TempVerts[tempVertsIndex].position.y += roundingOffset.y;
            if (tempVertsIndex == 3)
                toFill.AddUIVertexQuad(m_TempVerts);
        }
    }
   ...
}
```

可以看到主要是通过字体相关设置配合TextGenerator来生成text的mesh。注意还有一个FontTexture改变的时候通知text rebuild的回调。

```c#
public void FontTextureChanged()
{
    ...
    cachedTextGenerator.Invalidate();
	...
    if (CanvasUpdateRegistry.IsRebuildingGraphics() || CanvasUpdateRegistry.IsRebuildingLayout())
        UpdateGeometry();
    else
        SetAllDirty();
}
```

这个回调是通过FontUpdateTracker调用的，其中维护了所有的字体信息与文本组件的映射。在文本修改Font或者OnEnable、OnDisable的时候会相应的组测和反注册。并在Font发生textureRebuilt的时候触发所有的text的FontTextureChanged。

```c#
static Dictionary<Font, HashSet<Text>> m_Tracked = new Dictionary<Font, HashSet<Text>>();
```

UI框架里面自带了一个文本描边的效果Shadow.cs，继承BaseMeshEffect，重写ModifyMesh。

```c#
public override void ModifyMesh(VertexHelper vh)
{
    if (!IsActive())
        return;

    var output = ListPool<UIVertex>.Get();
    vh.GetUIVertexStream(output);

    ApplyShadow(output, effectColor, 0, output.Count, effectDistance.x, effectDistance.y);
    vh.Clear();
    vh.AddUIVertexTriangleStream(output);
    ListPool<UIVertex>.Release(output);
}
```

这儿为了避免描边的时候反复resize，把vh里面的数据取出来，后面直接设置capability。

```c#
protected void ApplyShadowZeroAlloc(List<UIVertex> verts, Color32 color, int start, int end, float x, float y)
{
    UIVertex vt;

    var neededCapacity = verts.Count + end - start;
    if (verts.Capacity < neededCapacity)
        verts.Capacity = neededCapacity;

    for (int i = start; i < end; ++i)
    {
        vt = verts[i];
        verts.Add(vt);

        Vector3 v = vt.position;
        v.x += x;
        v.y += y;
        vt.position = v;
        var newColor = color;
        if (m_UseGraphicAlpha)
            newColor.a = (byte)((newColor.a * verts[i].color.a) / 255);
        vt.color = newColor;
        verts[i] = vt;
    }
}
```

可以看出来是直接offset了一个quad加入到vertexs中。效果差，并且有一定开销。已经有很成熟的文本空间TMP了，导出SDF字体，字体放大缩小的时候不会糊，同时字体的各种描边效果都很容易实现。