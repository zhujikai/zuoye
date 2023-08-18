/// 一个可以实现点击显示的气泡

class JPopupMenuButtonItem {
  /// 按钮标题
  final String title;

  /// 点击响应
  final VoidCallback tap;

  JPopupMenuButtonItem({
    required this.title,
    required this.tap,
  });
}

class JPopupMenuButton extends StatelessWidget {
  const JPopupMenuButton(
      {Key? key, required this.child, required this.moreButtons, this.onTap})
      : super(key: key);

  final Widget child;

  /// 更多按钮展开的按钮列表
  final List<JPopupMenuButtonItem> moreButtons;

  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context) {
    TextStyle ts = JTextStyle4gray59Bold(13.sp);

    double maxWidth = 0;
    for (var element in moreButtons) {
      double width = element.title.computeParagraphSize(textStyle: ts).width;
      maxWidth = max(maxWidth, width);
    }

    return JPopupWidgetButton(
      child: child,
      popWidget: Builder(
        builder: (BuildContext context) {
          return ListView.builder(
            padding: EdgeInsets.zero,
            itemBuilder: (BuildContext context, int index) {
              return GestureDetector(
                behavior: HitTestBehavior.opaque,
                onTap: () {
                  Get.back();
                  onTap?.call();
                  moreButtons[index].tap.call();
                },
                child: Container(
                  height: 34.w,
                  padding: EdgeInsets.only(left: 12.w, top: 10.w),
                  child: JText(
                    moreButtons[index].title,
                    style: ts,
                  ),
                ),
              );
            },
            itemCount: moreButtons.length,
          );
        },
      ),
      popWidgetSize: Size(maxWidth + 24.w, 36.w * moreButtons.length),
    );
  }
}

class JPopupWidgetButton extends StatelessWidget {
  JPopupWidgetButton({
    Key? key,
    required this.child,
    required this.popWidget,
    required this.popWidgetSize,
    this.topPriority = true,
  }) : super(key: key);

  /// 按钮包裹内容
  final Widget child;

  /// 气泡内容
  final Builder popWidget;

  /// 预设大小,实际大小如果大于预设大小则按照实际大小自适应
  final Size popWidgetSize;

  /// 是否优先显示在上边
  final bool topPriority;

  final GlobalKey globalKey = GlobalKey();

  /// 点击按钮触发，用来计算点击位置，确定气泡箭头
  void _onAfterRending(BuildContext context) {
    RenderObject? renderObject = globalKey.currentContext?.findRenderObject();
    if (renderObject != null) {
      Size size = renderObject.paintBounds.size;
      var vector3 = renderObject.getTransformTo(null).getTranslation();
      Frame frame = Frame(vector3.x, vector3.y, size.width, size.height);

      showDialog(
        context: context,
        useSafeArea: false,
        barrierColor: Colors.transparent,
        builder: (context) {
          return _PopupContentView(
            touchWidgetFrame: frame,
            popWidget: popWidget,
            popWidgetSize: popWidgetSize,
            topPriority: topPriority,
          );
        },
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.transparent,
      child: GestureDetector(
        key: globalKey,
        behavior: HitTestBehavior.opaque,
        onTap: () {
          _onAfterRending(context);
        },
        child: child,
      ),
    );
  }
}

class Frame {
  final double x;
  final double y;
  final double width;
  final double height;
  const Frame(this.x, this.y, this.width, this.height);
}

class _PopupContentView extends StatefulWidget {
  const _PopupContentView({
    Key? key,
    required this.touchWidgetFrame,
    required this.popWidget,
    required this.popWidgetSize,
    this.topPriority = true,
  }) : super(key: key);

  final Frame touchWidgetFrame;
  final Builder popWidget;
  final Size popWidgetSize;
  final bool topPriority;

  @override
  State<_PopupContentView> createState() => _PopupContentViewState();
}

class _PopupContentViewState extends State<_PopupContentView> {
  final GlobalKey globalKey = GlobalKey();
  late final Widget contentWidget;

  late bool showTop;
  late double contentLeft;
  late double contentTop;

  Size? _popWidgetSize;

  get touchWidgetFrame => widget.touchWidgetFrame;
  Builder get popWidget => widget.popWidget;
  get popWidgetSize => _popWidgetSize ?? widget.popWidgetSize;
  get topPriority => widget.topPriority;

  @override
  void initState() {
    super.initState();

    // 计算布局
    _computeFrame();

    // 设置布局监听
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _onAfterRending(context);
    });

    // 获取气泡里面组件
    Widget builderWidget = popWidget.builder(context);

    // 判断如果是滚动组件，就不再重新布局
    if (builderWidget is ScrollView) {
      contentWidget = popWidget;
    } else {
      contentWidget = Column(
        children: [
          Container(
            key: globalKey,
            child: popWidget,
          ),
        ],
      );
    }
  }

  void _computeFrame() {
    // 展示在下面还是上面
    showTop = isShowTop(touchWidgetFrame);
    // 内容区域 y
    contentTop = computeContentTop(touchWidgetFrame, showTop);
    // 内容区域x 位置，先判断按箭头居中是否符合要求，如果不符合要求再进行调整
    contentLeft = computeContentLeft(touchWidgetFrame, showTop);
  }

  /// 第一次布局成功后，计算是否需要重新布局
  void _onAfterRending(BuildContext context) {
    RenderObject? renderObject = globalKey.currentContext?.findRenderObject();
    if (renderObject != null) {
      Size size = renderObject.paintBounds.size;
      if (size.height > popWidgetSize.height ||
          size.width > popWidgetSize.width) {
        _popWidgetSize = Size(max(size.width + 1, widget.popWidgetSize.width),
            max(size.height + 1, widget.popWidgetSize.height));
        _computeFrame();
        setState(() {});
      }
    }
  }

  /// 计算是否显示在按钮上边
  bool isShowTop(Frame frame) {
    bool showTop = topPriority;
    double topEdge =
        frame.y - 10 - popWidgetSize.height - ScreenUtil().statusBarHeight;
    double bottomEdge = (ScreenUtil().screenHeight -
            (frame.y + frame.height + 10 + popWidgetSize.height)) -
        ScreenUtil().bottomBarHeight;
    if (topEdge < 0 && bottomEdge < 0) {
      showTop = topEdge > bottomEdge;
    } else {
      if (showTop) {
        if (topEdge < 0) {
          showTop = false;
        }
      } else {
        if (bottomEdge < 0) {
          showTop = true;
        }
      }
    }
    return showTop;
  }

  /// 计算气泡顶部
  double computeContentTop(Frame frame, bool showTop) {
    double contentTop;
    if (showTop) {
      contentTop = frame.y - 10 - popWidgetSize.height;
    } else {
      contentTop = frame.y + frame.height + 10;
    }
    return contentTop;
  }

  /// 计算气泡左边
  double computeContentLeft(Frame frame, bool showTop) {
    double contentLeft =
        frame.x + (frame.width / 2) - (popWidgetSize.width / 2);
    if (contentLeft < 12 ||
        (ScreenUtil().screenWidth - contentLeft - popWidgetSize.width) < 12) {
      if (popWidgetSize.width > ScreenUtil().screenWidth) {
        contentLeft = 0;
      } else if (popWidgetSize.width > (ScreenUtil().screenWidth - 24)) {
        contentLeft = (ScreenUtil().screenWidth - popWidgetSize.width) / 2;
      } else if (contentLeft < 12) {
        contentLeft = 12;
      } else {
        contentLeft = ScreenUtil().screenWidth - popWidgetSize.width - 12;
      }
    }
    return contentLeft;
  }

  @override
  Widget build(BuildContext context) {
    IconData icon;
    double arrowTop, arrowLeft, shadowOffsetY;

    arrowLeft = touchWidgetFrame.x + (touchWidgetFrame.width / 2) - 25;
    if (showTop) {
      icon = Icons.arrow_drop_down_rounded;
      arrowTop = touchWidgetFrame.y - 33 - 1;
      shadowOffsetY = 3;
    } else {
      icon = Icons.arrow_drop_up_rounded;
      arrowTop = touchWidgetFrame.y + touchWidgetFrame.height - 17 + 1;
      shadowOffsetY = -3;
    }

    return SizedBox(
      height: ScreenUtil().screenHeight,
      width: ScreenUtil().screenWidth,
      child: Stack(
        children: [
          GestureDetector(
            onTap: () {
              Get.back();
            },
            onPanStart: (DragStartDetails details) {
              Get.back();
            },
            child: Container(
              width: ScreenUtil().screenWidth,
              height: ScreenUtil().screenHeight,
              color: Colors.transparent,
            ),
          ),
          Positioned(
            child: Container(
              alignment: Alignment.center,
              decoration: BoxDecoration(
                color: Colors.white,
                borderRadius: BorderRadius.circular(5),
                border: Border.all(
                  color: JColors.divider1,
                  width: 0.5,
                ),
                boxShadow: [
                  BoxShadow(
                    blurRadius: 12,
                    spreadRadius: 1,
                    color: Colors.black12,
                    offset: Offset(0, (showTop ? 4 : -4)),
                  ),
                ],
              ),
              child: contentWidget,
            ),
            height: popWidgetSize.height,
            width: popWidgetSize.width,
            left: contentLeft,
            top: contentTop,
          ),
          Positioned(
            child: IgnorePointer(
              child: Icon(
                icon,
                color: Colors.white,
                size: 50,
                shadows: [
                  BoxShadow(
                    blurRadius: 10,
                    spreadRadius: 1,
                    color: Colors.black12,
                    offset: Offset(0, shadowOffsetY),
                  ),
                ],
              ),
            ),
            width: 50,
            height: 50,
            top: arrowTop,
            left: arrowLeft,
          ),
        ],
      ),
    );
  }
}
