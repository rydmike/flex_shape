# flex_shape - Claude Opus 4.5 chat and plan

Make a Flutter package called FlexShape, that enables using BoxDecoration shape styling features to components like ElevatedButton, Card, TextField and Dialogs. These components cannot use the more elaborate styling that BoxDecoration offers and severely limit the way we can style them. We are limited to ShapeBorder.

Examine if the there is way to do this, also so that it can be used in theming of typical Material components that use ShapeBorder, often they have their own implementations of it.

The ShapeDecoration, may offer some help or ideas. If you find a way that works, make a plan for its implementation.

We are interested in offering gradients, and the elaborate inner/outer shadows (instead of elevation physics based shadows used in default Material). Capability to also use different corner types like RoundedRectangleBorder, RoundedSuperellipseBorder, ContinuousRectangleBorder and BeveledRectangleBorder is needed. We should also be able to use different type and radius for each corner.

The new shape should support shape animation and transitions to other ShapeBorders.

In the plan include a plan an example application that demonstrate the usage on buttons, Card, TextField Dialogs and other relevant shapes. Showcasing the different feature of the FlexShape used on components and the themes in Material ThemeData with their component themes.


# FlexShape Package Analysis and Implementation Plan

## Executive Summary

After analyzing Flutter's decoration and shape systems, I've identified a viable approach to bring `BoxDecoration`-like styling (gradients, multiple shadows) to Material components that use `ShapeBorder`. The key insight is that while we cannot directly use `BoxDecoration` with these components, we can create a hybrid system that:

1. **Extends `ShapeBorder`** to carry decoration data (gradients, shadows)
2. **Provides wrapper widgets** that render the full decoration while maintaining Material component behavior
3. **Integrates with theming** through custom component themes

---

## Technical Analysis

### The Core Problem

Material components like `ElevatedButton`, `Card`, `TextField`, and `Dialog` use `ShapeBorder` for their shape, which only defines:
- The outline path (`getOuterPath`, `getInnerPath`)
- Border painting (`paint` method)
- Animation interpolation (`lerpFrom`, `lerpTo`)

`BoxDecoration` offers much more:
- Gradient fills (linear, radial, sweep)
- Multiple box shadows with spread, blur, and color
- Background images
- Border with gradient support

### Why Direct Integration Is Impossible

Material components internally use `ShapeBorder` in specific ways:
- `Material` widget uses it for ink splash clipping
- `InkWell` uses the shape for ripple effects
- Components expect `ShapeBorder` for their `shape` parameters

### The Solution: FlexShape Architecture

The solution involves three layers:

1. **`FlexShapeBorder`**: A `ShapeBorder` subclass that carries decoration metadata and supports all corner types
2. **`FlexShapeDecoration`**: A `Decoration` class (like `ShapeDecoration`) that renders the full visual
3. **Wrapper Widgets**: `FlexButton`, `FlexCard`, `FlexTextField`, `FlexDialog` that compose the decoration with Material components

---

## Implementation Plan

### Phase 1: Core Shape System

#### 1.1 FlexCornerShape - Universal Corner Support

```dart
/// Defines a corner with type and radius
class FlexCorner {
  final FlexCornerType type;
  final double radius;
  
  const FlexCorner({
    this.type = FlexCornerType.rounded,
    this.radius = 0,
  });
}

enum FlexCornerType {
  rounded,      // Standard rounded (circular arc)
  superellipse, // iOS-style continuous curve
  continuous,   // ContinuousRectangleBorder style
  beveled,      // Cut corner
  sharp,        // No rounding
}

/// Four corners with independent types and radii
class FlexCorners {
  final FlexCorner topLeft;
  final FlexCorner topRight;
  final FlexCorner bottomLeft;
  final FlexCorner bottomRight;
  
  const FlexCorners({
    this.topLeft = const FlexCorner(),
    this.topRight = const FlexCorner(),
    this.bottomLeft = const FlexCorner(),
    this.bottomRight = const FlexCorner(),
  });
  
  /// Convenience constructors
  const FlexCorners.all(FlexCorner corner) : ...;
  const FlexCorners.symmetric({FlexCorner? vertical, FlexCorner? horizontal}) : ...;
}
```

#### 1.2 FlexShadow - Enhanced Shadow Model

```dart
class FlexShadow {
  final Color color;
  final double blurRadius;
  final double spreadRadius;
  final Offset offset;
  final bool isInner; // Inner shadow support!
  
  const FlexShadow({
    this.color = const Color(0x33000000),
    this.blurRadius = 0,
    this.spreadRadius = 0,
    this.offset = Offset.zero,
    this.isInner = false,
  });
  
  static FlexShadow lerp(FlexShadow? a, FlexShadow? b, double t);
}
```

#### 1.3 FlexShapeBorder - The Core ShapeBorder

```dart
class FlexShapeBorder extends ShapeBorder {
  final FlexCorners corners;
  final BorderSide side;
  
  // Decoration data carried by the shape
  final Gradient? gradient;
  final Color? color;
  final List<FlexShadow> shadows;
  final DecorationImage? image;
  
  const FlexShapeBorder({
    this.corners = const FlexCorners(),
    this.side = BorderSide.none,
    this.gradient,
    this.color,
    this.shadows = const [],
    this.image,
  });
  
  @override
  Path getOuterPath(Rect rect, {TextDirection? textDirection}) {
    return _buildPath(rect, corners);
  }
  
  @override
  Path getInnerPath(Rect rect, {TextDirection? textDirection}) {
    return _buildPath(rect.deflate(side.width), corners);
  }
  
  @override
  void paint(Canvas canvas, Rect rect, {TextDirection? textDirection}) {
    // Only paints the border - full decoration is handled by FlexShapeDecoration
    if (side.style != BorderStyle.none) {
      canvas.drawPath(getOuterPath(rect), side.toPaint());
    }
  }
  
  @override
  ShapeBorder? lerpFrom(ShapeBorder? a, double t) { ... }
  
  @override
  ShapeBorder? lerpTo(ShapeBorder? b, double t) { ... }
  
  /// Convert to standard Flutter ShapeBorder for Material components
  ShapeBorder toMaterialShape() {
    // Returns appropriate RoundedRectangleBorder, etc.
    // based on corner configuration
  }
}
```

### Phase 2: Decoration System

#### 2.1 FlexShapeDecoration

```dart
class FlexShapeDecoration extends Decoration {
  final FlexShapeBorder shape;
  final Gradient? gradient;
  final Color? color;
  final List<FlexShadow> shadows;
  final DecorationImage? image;
  
  const FlexShapeDecoration({
    required this.shape,
    this.gradient,
    this.color,
    this.shadows = const [],
    this.image,
  });
  
  /// Create from a FlexShapeBorder (extracts its decoration data)
  factory FlexShapeDecoration.fromShape(FlexShapeBorder shape) {
    return FlexShapeDecoration(
      shape: shape,
      gradient: shape.gradient,
      color: shape.color,
      shadows: shape.shadows,
      image: shape.image,
    );
  }
  
  @override
  BoxPainter createBoxPainter([VoidCallback? onChanged]) {
    return _FlexShapePainter(this, onChanged);
  }
  
  @override
  FlexShapeDecoration? lerpFrom(Decoration? a, double t) { ... }
  
  @override
  FlexShapeDecoration? lerpTo(Decoration? b, double t) { ... }
}

class _FlexShapePainter extends BoxPainter {
  @override
  void paint(Canvas canvas, Offset offset, ImageConfiguration configuration) {
    final rect = offset & configuration.size!;
    final path = shape.getOuterPath(rect);
    
    // 1. Paint outer shadows
    for (final shadow in shadows.where((s) => !s.isInner)) {
      _paintOuterShadow(canvas, path, shadow);
    }
    
    // 2. Paint fill (color or gradient)
    if (gradient != null) {
      final paint = Paint()..shader = gradient!.createShader(rect);
      canvas.drawPath(path, paint);
    } else if (color != null) {
      canvas.drawPath(path, Paint()..color = color!);
    }
    
    // 3. Paint image
    if (image != null) {
      _paintImage(canvas, rect, path);
    }
    
    // 4. Paint inner shadows (clipped to path)
    canvas.save();
    canvas.clipPath(path);
    for (final shadow in shadows.where((s) => s.isInner)) {
      _paintInnerShadow(canvas, rect, path, shadow);
    }
    canvas.restore();
    
    // 5. Paint border
    shape.paint(canvas, rect);
  }
  
  void _paintInnerShadow(Canvas canvas, Rect rect, Path path, FlexShadow shadow) {
    // Inner shadow technique: draw inverted shadow outside the path
    // then clip to show only the inner portion
    final shadowPaint = Paint()
      ..color = shadow.color
      ..maskFilter = MaskFilter.blur(BlurStyle.normal, shadow.blurRadius);
    
    // Create a larger rect and subtract the path
    final outerRect = rect.inflate(shadow.blurRadius * 2 + shadow.spreadRadius);
    final outerPath = Path()..addRect(outerRect);
    
    // Offset and shrink the inner path
    final innerPath = path.shift(shadow.offset);
    // Apply spread (shrink for inner shadow)
    
    final shadowPath = Path.combine(PathOperation.difference, outerPath, innerPath);
    canvas.drawPath(shadowPath, shadowPaint);
  }
}
```

### Phase 3: Wrapper Widgets

#### 3.1 FlexContainer - Base Decorated Container

```dart
class FlexContainer extends StatelessWidget {
  final FlexShapeBorder shape;
  final Widget? child;
  final double? width;
  final double? height;
  final EdgeInsetsGeometry? padding;
  final EdgeInsetsGeometry? margin;
  
  const FlexContainer({
    super.key,
    required this.shape,
    this.child,
    this.width,
    this.height,
    this.padding,
    this.margin,
  });
  
  @override
  Widget build(BuildContext context) {
    return Container(
      width: width,
      height: height,
      margin: margin,
      decoration: FlexShapeDecoration.fromShape(shape),
      child: Padding(
        padding: padding ?? EdgeInsets.zero,
        child: child,
      ),
    );
  }
}
```

#### 3.2 FlexButton - ElevatedButton with FlexShape

```dart
class FlexButton extends StatelessWidget {
  final FlexShapeBorder shape;
  final VoidCallback? onPressed;
  final Widget child;
  final bool useInkWell; // Option to use ink effects
  
  const FlexButton({
    super.key,
    required this.shape,
    this.onPressed,
    required this.child,
    this.useInkWell = true,
  });
  
  @override
  Widget build(BuildContext context) {
    final decoration = FlexShapeDecoration.fromShape(shape);
    
    return Semantics(
      button: true,
      child: DecoratedBox(
        decoration: decoration,
        child: Material(
          type: MaterialType.transparency,
          shape: shape.toMaterialShape(),
          child: InkWell(
            onTap: onPressed,
            customBorder: shape.toMaterialShape(),
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
              child: child,
            ),
          ),
        ),
      ),
    );
  }
}

/// Animated version with implicit animation
class AnimatedFlexButton extends ImplicitlyAnimatedWidget {
  final FlexShapeBorder shape;
  final VoidCallback? onPressed;
  final Widget child;
  
  const AnimatedFlexButton({
    super.key,
    required this.shape,
    this.onPressed,
    required this.child,
    super.duration = const Duration(milliseconds: 200),
    super.curve = Curves.easeInOut,
  });
  
  @override
  AnimatedWidgetBaseState<AnimatedFlexButton> createState() => 
      _AnimatedFlexButtonState();
}

class _AnimatedFlexButtonState extends AnimatedWidgetBaseState<AnimatedFlexButton> {
  FlexShapeBorderTween? _shapeTween;
  
  @override
  void forEachTween(TweenVisitor<dynamic> visitor) {
    _shapeTween = visitor(
      _shapeTween,
      widget.shape,
      (value) => FlexShapeBorderTween(begin: value as FlexShapeBorder),
    ) as FlexShapeBorderTween?;
  }
  
  @override
  Widget build(BuildContext context) {
    return FlexButton(
      shape: _shapeTween!.evaluate(animation),
      onPressed: widget.onPressed,
      child: widget.child,
    );
  }
}
```

#### 3.3 FlexCard

```dart
class FlexCard extends StatelessWidget {
  final FlexShapeBorder shape;
  final Widget? child;
  final EdgeInsetsGeometry? padding;
  final Clip clipBehavior;
  
  const FlexCard({
    super.key,
    required this.shape,
    this.child,
    this.padding,
    this.clipBehavior = Clip.antiAlias,
  });
  
  @override
  Widget build(BuildContext context) {
    return DecoratedBox(
      decoration: FlexShapeDecoration.fromShape(shape),
      child: ClipPath(
        clipper: _ShapeClipper(shape),
        clipBehavior: clipBehavior,
        child: Material(
          type: MaterialType.transparency,
          child: Padding(
            padding: padding ?? const EdgeInsets.all(16),
            child: child,
          ),
        ),
      ),
    );
  }
}
```

#### 3.4 FlexTextField

```dart
class FlexTextField extends StatelessWidget {
  final FlexShapeBorder shape;
  final TextEditingController? controller;
  final String? hintText;
  final InputDecoration? decoration;
  // ... other TextField parameters
  
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        // Background decoration
        Positioned.fill(
          child: DecoratedBox(
            decoration: FlexShapeDecoration.fromShape(shape),
          ),
        ),
        // The actual TextField with transparent decoration
        TextField(
          controller: controller,
          decoration: (decoration ?? const InputDecoration()).copyWith(
            filled: false,
            border: InputBorder.none,
            enabledBorder: InputBorder.none,
            focusedBorder: InputBorder.none,
            hintText: hintText,
          ),
        ),
      ],
    );
  }
}
```

#### 3.5 FlexDialog

```dart
class FlexDialog extends StatelessWidget {
  final FlexShapeBorder shape;
  final Widget? title;
  final Widget? content;
  final List<Widget>? actions;
  
  @override
  Widget build(BuildContext context) {
    return Dialog(
      backgroundColor: Colors.transparent,
      shape: shape.toMaterialShape(),
      child: DecoratedBox(
        decoration: FlexShapeDecoration.fromShape(shape),
        child: ClipPath(
          clipper: _ShapeClipper(shape),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              if (title != null) 
                Padding(
                  padding: const EdgeInsets.all(24),
                  child: title,
                ),
              if (content != null)
                Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 24),
                  child: content,
                ),
              if (actions != null)
                Padding(
                  padding: const EdgeInsets.all(8),
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.end,
                    children: actions!,
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }
}

/// Show a FlexDialog
Future<T?> showFlexDialog<T>({
  required BuildContext context,
  required FlexShapeBorder shape,
  Widget? title,
  Widget? content,
  List<Widget>? actions,
}) {
  return showDialog<T>(
    context: context,
    builder: (context) => FlexDialog(
      shape: shape,
      title: title,
      content: content,
      actions: actions,
    ),
  );
}
```

### Phase 4: Animation Support

#### 4.1 FlexShapeBorderTween

```dart
class FlexShapeBorderTween extends Tween<FlexShapeBorder> {
  FlexShapeBorderTween({super.begin, super.end});
  
  @override
  FlexShapeBorder lerp(double t) {
    if (begin == null && end == null) return const FlexShapeBorder();
    if (begin == null) return end!;
    if (end == null) return begin!;
    
    return FlexShapeBorder(
      corners: FlexCorners.lerp(begin!.corners, end!.corners, t),
      side: BorderSide.lerp(begin!.side, end!.side, t),
      gradient: Gradient.lerp(begin!.gradient, end!.gradient, t),
      color: Color.lerp(begin!.color, end!.color, t),
      shadows: FlexShadow.lerpList(begin!.shadows, end!.shadows, t),
    );
  }
}

extension on FlexCorners {
  static FlexCorners lerp(FlexCorners a, FlexCorners b, double t) {
    return FlexCorners(
      topLeft: FlexCorner.lerp(a.topLeft, b.topLeft, t),
      topRight: FlexCorner.lerp(a.topRight, b.topRight, t),
      bottomLeft: FlexCorner.lerp(a.bottomLeft, b.bottomLeft, t),
      bottomRight: FlexCorner.lerp(a.bottomRight, b.bottomRight, t),
    );
  }
}
```

#### 4.2 Animated Decoration

```dart
class AnimatedFlexDecoration extends ImplicitlyAnimatedWidget {
  final FlexShapeDecoration decoration;
  final Widget child;
  
  const AnimatedFlexDecoration({
    super.key,
    required this.decoration,
    required this.child,
    super.duration = const Duration(milliseconds: 200),
    super.curve = Curves.easeInOut,
  });
  
  @override
  AnimatedWidgetBaseState<AnimatedFlexDecoration> createState() =>
      _AnimatedFlexDecorationState();
}
```

### Phase 5: Theme Integration

#### 5.1 FlexShapeTheme

```dart
class FlexShapeThemeData {
  final FlexShapeBorder? buttonShape;
  final FlexShapeBorder? cardShape;
  final FlexShapeBorder? dialogShape;
  final FlexShapeBorder? inputShape;
  final FlexShapeBorder? chipShape;
  
  const FlexShapeThemeData({
    this.buttonShape,
    this.cardShape,
    this.dialogShape,
    this.inputShape,
    this.chipShape,
  });
  
  static FlexShapeThemeData of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<FlexShapeTheme>()?.data 
        ?? const FlexShapeThemeData();
  }
  
  FlexShapeThemeData copyWith({...});
  
  static FlexShapeThemeData lerp(FlexShapeThemeData a, FlexShapeThemeData b, double t);
}

class FlexShapeTheme extends InheritedWidget {
  final FlexShapeThemeData data;
  
  const FlexShapeTheme({
    super.key,
    required this.data,
    required super.child,
  });
  
  @override
  bool updateShouldNotify(FlexShapeTheme oldWidget) => data != oldWidget.data;
}
```

#### 5.2 Preset Themes

```dart
class FlexShapePresets {
  /// Soft, rounded corners with subtle shadows
  static FlexShapeThemeData soft({Color? primary}) => FlexShapeThemeData(
    buttonShape: FlexShapeBorder(
      corners: FlexCorners.all(const FlexCorner(
        type: FlexCornerType.superellipse,
        radius: 12,
      )),
      gradient: LinearGradient(
        colors: [primary ?? Colors.blue, (primary ?? Colors.blue).withOpacity(0.8)],
      ),
      shadows: [
        FlexShadow(
          color: (primary ?? Colors.blue).withOpacity(0.3),
          blurRadius: 8,
          offset: const Offset(0, 4),
        ),
      ],
    ),
    // ... other shapes
  );
  
  /// Sharp, modern look with inner shadows
  static FlexShapeThemeData sharp({Color? primary}) => FlexShapeThemeData(
    buttonShape: FlexShapeBorder(
      corners: FlexCorners.all(const FlexCorner(
        type: FlexCornerType.beveled,
        radius: 8,
      )),
      color: primary ?? Colors.blue,
      shadows: [
        FlexShadow(
          color: Colors.black.withOpacity(0.5),
          blurRadius: 4,
          isInner: true,
          offset: const Offset(2, 2),
        ),
        FlexShadow(
          color: Colors.white.withOpacity(0.2),
          blurRadius: 4,
          isInner: true,
          offset: const Offset(-2, -2),
        ),
      ],
    ),
    // ... other shapes
  );
  
  /// iOS-style continuous corners
  static FlexShapeThemeData ios({Color? primary}) => FlexShapeThemeData(
    buttonShape: FlexShapeBorder(
      corners: FlexCorners.all(const FlexCorner(
        type: FlexCornerType.superellipse,
        radius: 10,
      )),
      color: primary ?? Colors.blue,
    ),
    // ... other shapes
  );
}
```

---

## Example Application Plan

### App Structure

```
example/
├── lib/
│   ├── main.dart
│   ├── pages/
│   │   ├── home_page.dart
│   │   ├── buttons_showcase.dart
│   │   ├── cards_showcase.dart
│   │   ├── inputs_showcase.dart
│   │   ├── dialogs_showcase.dart
│   │   ├── corners_showcase.dart
│   │   ├── shadows_showcase.dart
│   │   ├── gradients_showcase.dart
│   │   ├── animations_showcase.dart
│   │   └── themes_showcase.dart
│   ├── widgets/
│   │   ├── showcase_scaffold.dart
│   │   ├── code_preview.dart
│   │   └── property_editor.dart
│   └── themes/
│       ├── soft_theme.dart
│       ├── sharp_theme.dart
│       └── ios_theme.dart
└── pubspec.yaml
```

### Example Pages Content

#### 1. Buttons Showcase (`buttons_showcase.dart`)

Demonstrates:
- Basic `FlexButton` with gradient
- Button with inner shadows (neumorphic style)
- Button with multiple outer shadows (glow effect)
- Mixed corner types (e.g., rounded top, beveled bottom)
- Hover/press state animations
- Comparison with standard `ElevatedButton`

```dart
// Example code shown in the showcase
FlexButton(
  shape: FlexShapeBorder(
    corners: FlexCorners.all(
      const FlexCorner(type: FlexCornerType.superellipse, radius: 16),
    ),
    gradient: const LinearGradient(
      colors: [Color(0xFF6366F1), Color(0xFF8B5CF6)],
      begin: Alignment.topLeft,
      end: Alignment.bottomRight,
    ),
    shadows: [
      FlexShadow(
        color: const Color(0xFF6366F1).withOpacity(0.4),
        blurRadius: 12,
        offset: const Offset(0, 6),
      ),
    ],
  ),
  onPressed: () {},
  child: const Text('Gradient Button', style: TextStyle(color: Colors.white)),
),
```

#### 2. Cards Showcase (`cards_showcase.dart`)

Demonstrates:
- Product cards with image and gradient overlay
- Neumorphic cards with inner/outer shadows
- Cards with different corner combinations
- Animated card hover effects
- Comparison with standard `Card`

#### 3. Inputs Showcase (`inputs_showcase.dart`)

Demonstrates:
- Styled `FlexTextField` with gradient border
- Search input with icon and shadows
- Text area with custom corners
- Form with themed inputs
- Focus state animations

#### 4. Dialogs Showcase (`dialogs_showcase.dart`)

Demonstrates:
- Alert dialog with gradient header
- Confirmation dialog with custom shadows
- Full-screen dialog with unique shape
- Dialog animations

#### 5. Corner Types Showcase (`corners_showcase.dart`)

Interactive demo showing:
- Side-by-side comparison of all corner types
- Per-corner type selection
- Per-corner radius slider
- Real-time preview on a card

#### 6. Shadows Showcase (`shadows_showcase.dart`)

Interactive demo showing:
- Outer shadows with adjustable blur, spread, offset
- Inner shadows for inset effects
- Multiple shadow layering
- Color picker for shadows
- Glow effects

#### 7. Gradients Showcase (`gradients_showcase.dart`)

Demonstrates:
- Linear gradients with direction control
- Radial gradients
- Sweep gradients
- Gradient stops editor
- Gradient animation

#### 8. Animations Showcase (`animations_showcase.dart`)

Demonstrates:
- Shape morphing (corner type transitions)
- Gradient animation
- Shadow animation
- Combined property animation
- Explicit vs implicit animation approaches

#### 9. Themes Showcase (`themes_showcase.dart`)

Demonstrates:
- Preset themes (Soft, Sharp, iOS)
- Custom theme creation
- Theme switching with animation
- Full app themed with `FlexShapeTheme`

---

## Package File Structure

```
flex_shape/
├── lib/
│   ├── flex_shape.dart                 # Main export file
│   ├── src/
│   │   ├── core/
│   │   │   ├── flex_corner.dart        # FlexCorner, FlexCornerType, FlexCorners
│   │   │   ├── flex_shadow.dart        # FlexShadow
│   │   │   └── flex_shape_border.dart  # FlexShapeBorder
│   │   ├── decoration/
│   │   │   ├── flex_shape_decoration.dart
│   │   │   └── flex_shape_painter.dart
│   │   ├── widgets/
│   │   │   ├── flex_container.dart
│   │   │   ├── flex_button.dart
│   │   │   ├── flex_card.dart
│   │   │   ├── flex_text_field.dart
│   │   │   ├── flex_dialog.dart
│   │   │   └── flex_chip.dart
│   │   ├── animation/
│   │   │   ├── flex_shape_tween.dart
│   │   │   ├── animated_flex_decoration.dart
│   │   │   └── animated_flex_widgets.dart
│   │   ├── theme/
│   │   │   ├── flex_shape_theme.dart
│   │   │   └── flex_shape_presets.dart
│   │   └── utils/
│   │       ├── path_builder.dart       # Builds paths for different corner types
│   │       └── shape_clipper.dart
├── example/
│   └── ... (as described above)
├── test/
│   ├── flex_corner_test.dart
│   ├── flex_shadow_test.dart
│   ├── flex_shape_border_test.dart
│   ├── flex_shape_decoration_test.dart
│   ├── animation_test.dart
│   └── widget_tests/
├── pubspec.yaml
├── README.md
├── CHANGELOG.md
└── LICENSE
```

---

## Implementation Timeline

| Phase | Description | Estimated Effort |
|-------|-------------|------------------|
| 1 | Core shape system (FlexCorner, FlexShadow, FlexShapeBorder) | 3-4 days |
| 2 | Decoration system (FlexShapeDecoration, painter) | 2-3 days |
| 3 | Wrapper widgets (Button, Card, TextField, Dialog) | 3-4 days |
| 4 | Animation support | 2-3 days |
| 5 | Theme integration | 1-2 days |
| 6 | Example application | 3-4 days |
| 7 | Documentation & testing | 2-3 days |

**Total: ~16-23 days**

---

## Key Technical Challenges & Solutions

### Challenge 1: Inner Shadows
**Solution**: Use path subtraction technique - draw shadow on inverted path, clip to original path.

### Challenge 2: Mixed Corner Types
**Solution**: Build path segment by segment, using appropriate arc/line for each corner type.

### Challenge 3: Superellipse Corners
**Solution**: Use cubic Bézier curves approximating the squircle formula, similar to `RoundedSuperellipseBorder`.

### Challenge 4: Animation Between Different Corner Types
**Solution**: Interpolate both radius and corner type "weight", morphing between path geometries.

### Challenge 5: Material Ink Effects
**Solution**: Layer the `FlexShapeDecoration` behind a transparent `Material` widget with the shape's outline.

---

## Conclusion

This approach provides a comprehensive solution for bringing `BoxDecoration`-level styling to Material components while:
- Maintaining Material Design behaviors (ink splashes, accessibility)
- Supporting all requested features (gradients, inner/outer shadows, corner types)
- Enabling smooth animations between states
- Integrating with Flutter's theming system

# Limits with above plan - review and revise

This solution may not be entirely in line with what is sought, we want to be able the provide and use the new FlexShape also in existing Flutter Material Components directly to their shape property, in components like Card, Dialog and ElevatedButton (inside the style that is a ButtonStyle) and also to their shape property in their existing? Can you verify and make sure that this is achieved with this plan?

# Revised plan

You raise an excellent point. Let me analyze whether `FlexShapeBorder` can be directly assigned to the `shape` property of existing Material components and their theme configurations.

## Analysis: Direct Shape Property Assignment

### Testing the Approach

The key question is: **Can a custom `ShapeBorder` subclass that carries decoration data (gradients, shadows) actually render those decorations when used in Material components?**

Let me trace through how Material components use `ShapeBorder`:

### How Material Components Use ShapeBorder

**Card:**
```dart
// Inside Card widget (simplified)
Material(
  shape: shape, // Our FlexShapeBorder would go here
  color: color,
  elevation: elevation,
  child: child,
)
```

**Material widget internally:**
```dart
// Material uses ShapeBorderDecoration which only uses:
// - shape.getOuterPath() for clipping
// - shape.paint() for border painting
// It does NOT use any custom properties we add
```

**ElevatedButton (via ButtonStyle):**
```dart
ButtonStyle(
  shape: MaterialStateProperty.all(ourFlexShapeBorder),
)
// Internally uses Material widget similarly
```

### The Core Problem

When you assign a `ShapeBorder` to a Material component's `shape` property:

1. **The `Material` widget controls the background painting** - it uses `color` and `elevation` properties, NOT anything from `ShapeBorder`
2. **`ShapeBorder.paint()` only paints the border** - not fills, gradients, or shadows
3. **The component ignores custom properties** - even if `FlexShapeBorder` carries gradient/shadow data, `Material` won't use them

Let me verify this by examining Flutter's source:I found a critical piece of information. Let me examine the `paintInterior` method more closely - this is a newer addition to `ShapeBorder` that might be the key:This is interesting but doesn't solve our problem. Let me look at how `Material` widget actually paints the interior:Let me search for the raw source:Now I see the `_ShapeBorderPaint` class is used internally. Let me search for how `paintInterior` is actually called by Material:Now I understand the key mechanism. The `paintInterior` method receives a `Paint` object from the Material widget, and it's supposed to use that Paint to draw the interior. This is the crucial limitation - **the Material widget controls the Paint, not the ShapeBorder**.

Let me verify by looking at how Material actually calls paintInterior:Now I have a clear picture. Looking at `RenderPhysicalShape.paint` shows that the Material widget internally creates its own `Paint()..color = color` to fill the shape, completely ignoring any paint information from the ShapeBorder.

## Verified Conclusion: Direct Assignment Will NOT Work for Gradients/Shadows

The analysis confirms:

1. **Material components control their own painting** - they use `PhysicalShape`/`RenderPhysicalShape` which creates `Paint()..color = color` internally
2. **ShapeBorder only provides the path** - via `getOuterPath()` for clipping and `paint()` for borders
3. **Even `paintInterior` doesn't help** - it receives a `Paint` object FROM the Material widget (with a solid color), it doesn't create its own

This means **you cannot simply assign a `FlexShapeBorder` with gradient/shadow data to `Card.shape` or `ElevatedButton.style.shape` and expect it to render those decorations**.

---

## Revised Plan: Two-Pronged Approach

Given this constraint, here's a revised, honest plan that addresses your requirements:

### Option A: Wrapper-Based Approach (Full Features, Slight API Difference)

This provides full `BoxDecoration`-like features but requires using wrapper widgets:

```dart
// Instead of:
Card(
  shape: flexShape, // Won't render gradients/shadows
  child: content,
)

// Use:
FlexCard(
  shape: flexShape, // Full gradient/shadow support
  child: content,
)
```

### Option B: Theme Extension + Background Layer Approach

This allows using standard components with a background decoration layer, integrated via theme extensions:

```dart
// Create theme with FlexShape backgrounds
ThemeData(
  extensions: [
    FlexShapeThemeExtension(
      cardDecoration: FlexShapeDecoration(...),
      buttonDecoration: FlexShapeDecoration(...),
    ),
  ],
)

// Standard Card, but wrapped automatically with decoration
FlexShapeScope(
  child: Card(...), // Gets FlexShape background from theme
)
```

### Option C: Pure ShapeBorder Features (Direct Assignment, Limited Features)

For features that **can** work via direct assignment (mixed corner types, custom border painting), we can create a `FlexShapeBorder` that works directly:

```dart
// This WORKS for shape/corners only (no gradients/shadows):
Card(
  shape: FlexShapeBorder(
    corners: FlexCorners(
      topLeft: FlexCorner(type: FlexCornerType.rounded, radius: 20),
      topRight: FlexCorner(type: FlexCornerType.beveled, radius: 12),
      bottomLeft: FlexCorner(type: FlexCornerType.superellipse, radius: 16),
      bottomRight: FlexCorner(type: FlexCornerType.continuous, radius: 8),
    ),
    side: BorderSide(color: Colors.blue, width: 2),
  ),
  child: content,
)

// Same in ButtonStyle:
ElevatedButton(
  style: ElevatedButton.styleFrom(
    shape: FlexShapeBorder(
      corners: FlexCorners.all(
        FlexCorner(type: FlexCornerType.superellipse, radius: 12),
      ),
    ),
  ),
  onPressed: () {},
  child: Text('Button'),
)
```

---

## Updated Implementation Plan

### Phase 1: Core `FlexShapeBorder` (Direct Assignment Compatible)

**Features that WORK via direct assignment:**
- ✅ Mixed corner types per corner
- ✅ Different radius per corner
- ✅ Custom border painting (solid, gradient borders)
- ✅ Animation/lerping between shapes
- ✅ All existing `OutlinedBorder` features

```dart
class FlexShapeBorder extends OutlinedBorder {
  final FlexCorners corners;
  
  const FlexShapeBorder({
    this.corners = const FlexCorners(),
    super.side,
  });
  
  @override
  Path getOuterPath(Rect rect, {TextDirection? textDirection}) {
    return _FlexPathBuilder.build(rect, corners);
  }
  
  @override
  Path getInnerPath(Rect rect, {TextDirection? textDirection}) {
    return _FlexPathBuilder.build(
      rect.deflate(side.strokeInset),
      corners,
    );
  }
  
  @override
  void paint(Canvas canvas, Rect rect, {TextDirection? textDirection}) {
    // Paint the border (this IS called by Material)
    if (side.style != BorderStyle.none) {
      final path = getOuterPath(rect);
      canvas.drawPath(path, side.toPaint());
    }
  }
  
  @override
  FlexShapeBorder copyWith({BorderSide? side, FlexCorners? corners}) {
    return FlexShapeBorder(
      side: side ?? this.side,
      corners: corners ?? this.corners,
    );
  }
  
  @override
  ShapeBorder? lerpFrom(ShapeBorder? a, double t) { /* ... */ }
  
  @override
  ShapeBorder? lerpTo(ShapeBorder? b, double t) { /* ... */ }
}
```

### Phase 2: `FlexShapeDecoration` (For Full Features)

For gradients, shadows, and images - use `FlexShapeDecoration` with a `Container` or wrapper:

```dart
class FlexShapeDecoration extends Decoration {
  final FlexShapeBorder shape;
  final Gradient? gradient;
  final Color? color;
  final List<FlexShadow> shadows; // Supports inner shadows
  final DecorationImage? image;
  
  // ... implementation
}
```

### Phase 3: Smart Wrapper Widgets

Widgets that combine decoration + Material behavior:

```dart
/// FlexCard - Full decoration support with Material ink effects
class FlexCard extends StatelessWidget {
  final FlexShapeDecoration decoration;
  final Widget? child;
  // ... other Card properties
  
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        // Background: Full decoration (gradients, shadows)
        Positioned.fill(
          child: DecoratedBox(decoration: decoration),
        ),
        // Foreground: Transparent Material for ink effects
        Material(
          type: MaterialType.transparency,
          shape: decoration.shape,
          clipBehavior: Clip.antiAlias,
          child: child,
        ),
      ],
    );
  }
}

/// FlexElevatedButton - Full decoration support  
class FlexElevatedButton extends StatelessWidget {
  final FlexShapeDecoration decoration;
  final VoidCallback? onPressed;
  final Widget child;
  
  @override
  Widget build(BuildContext context) {
    return DecoratedBox(
      decoration: decoration,
      child: Material(
        type: MaterialType.transparency,
        shape: decoration.shape,
        child: InkWell(
          onTap: onPressed,
          customBorder: decoration.shape,
          child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: child,
          ),
        ),
      ),
    );
  }
}
```

### Phase 4: Theme Integration

```dart
// Theme extension for FlexShape defaults
class FlexShapeThemeExtension extends ThemeExtension<FlexShapeThemeExtension> {
  final FlexShapeBorder? defaultButtonShape;
  final FlexShapeBorder? defaultCardShape;
  final FlexShapeDecoration? buttonDecoration;
  final FlexShapeDecoration? cardDecoration;
  
  // ... lerp, copyWith
}

// Usage in theme
MaterialApp(
  theme: ThemeData(
    // Standard Material theming - shapes only (no gradients)
    cardTheme: CardTheme(
      shape: FlexShapeBorder(
        corners: FlexCorners.all(
          FlexCorner(type: FlexCornerType.superellipse, radius: 16),
        ),
      ),
    ),
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        shape: FlexShapeBorder(
          corners: FlexCorners.all(
            FlexCorner(type: FlexCornerType.continuous, radius: 12),
          ),
        ),
      ),
    ),
    // FlexShape extension for full decoration features
    extensions: [
      FlexShapeThemeExtension(
        cardDecoration: FlexShapeDecoration(
          shape: FlexShapeBorder(...),
          gradient: LinearGradient(...),
          shadows: [...],
        ),
      ),
    ],
  ),
)
```

---

## Summary: What Works Where

| Feature | Direct to `shape` property | Via Wrapper/Decoration |
|---------|----------------------------|------------------------|
| Mixed corner types | ✅ Yes | ✅ Yes |
| Different radius per corner | ✅ Yes | ✅ Yes |
| Border with solid color | ✅ Yes | ✅ Yes |
| Shape animation/lerping | ✅ Yes | ✅ Yes |
| **Gradient fill** | ❌ No | ✅ Yes |
| **Multiple shadows** | ❌ No | ✅ Yes |
| **Inner shadows** | ❌ No | ✅ Yes |
| **Background image** | ❌ No | ✅ Yes |
| Material ink effects | ✅ Automatic | ✅ Via transparent Material |

---

## Honest Assessment

**The core Flutter architecture does not allow ShapeBorder to control the fill/decoration of Material components.** This is a fundamental design decision in Flutter's Material library.

Your options are:
1. **Accept the limitation** - Use `FlexShapeBorder` for corner types/borders directly in Material components, and wrapper widgets for gradients/shadows
2. **Request Flutter team changes** - File a feature request for a "Decoration-aware ShapeBorder"
3. **Fork/extend Material widgets** - Create custom versions of Card, Button, etc. that accept decorations

Would you like me to proceed with this revised plan, or explore alternative approaches?

# Go with option 2 - Flutter change!

For the solution to offer the desired benefits of increasing the styling and theming flexibility of Flutter's existing Material widgets it looks like option 2 is the only way forward. 

Formulate a request that is in line with Flutter feature request guidelines that describes this as a new feature request. List the sought features and benefits. Analyze impact on Flutter's APIs and suggest needed changes in the framework to support decoration aware ShapeBorder. 

The suggestion should also include the new above discussed FlexShapeBorder using ShapeBorder implementation, with all its corner and corner type, shape lerping features. We should call it something else than FlexShapeBorder, DecorationShapeBorder would perhaps be more inline with Flutter API naming. 

Make it a clear well styled proposal, that explains what problem it solves and benefits the solution would bring. Include a high level summary approach for how it could be solved, with a more detailed proposal after it.


# Feature Request: Decoration-Aware ShapeBorder for Material Components

## Summary

Enable Material components (`Card`, `Dialog`, `TextField`, `Chip`, etc.) to render full decoration styling (gradients, multiple shadows, inner shadows, images) through their existing `shape` property by introducing a new `ShapeBorder` subclass that carries decoration data and modifying the `Material` widget to render it.

---

## Use Case

### The Problem

Flutter's `BoxDecoration` offers rich styling capabilities that developers frequently need:
- **Gradient fills** (linear, radial, sweep)
- **Multiple box shadows** with spread, blur, and offset
- **Inner shadows** (inset shadows)
- **Background images**
- **Custom shadow colors** (beyond elevation-based shadows)

However, **Material components cannot use `BoxDecoration`**. Components like `Card`, `Dialog`, `ElevatedButton`, `TextField`, `Chip`, and others use `ShapeBorder` for their shape, which only defines:
- The outline path (`getOuterPath`, `getInnerPath`)
- Border painting (`paint` method)
- Animation interpolation (`lerpFrom`, `lerpTo`)

The `Material` widget internally controls background painting using its own `color` and `elevation` properties, completely ignoring any custom properties a `ShapeBorder` subclass might carry.

### Current Workarounds Are Inadequate

Developers currently resort to:

1. **Wrapper stacks** - Wrapping components in `Stack` with `DecoratedBox`:
   ```dart
   Stack(
     children: [
       DecoratedBox(decoration: myGradientDecoration),
       Card(color: Colors.transparent, child: content),
     ],
   )
   ```
    - Breaks widget tree semantics
    - Complicates layout
    - Doesn't integrate with theming

2. **Custom widget implementations** - Recreating Material components:
    - Loses Material behaviors (ink splash, focus handling, accessibility)
    - Significant development overhead
    - Diverges from framework updates

3. **`ButtonStyle.backgroundBuilder`** (Flutter 3.22+):
    - Only works for button components
    - Requires builder function instead of declarative style
    - Cannot be used in `ThemeData` component themes

### Real-World Impact

This limitation affects common design requirements:
- **Gradient buttons** - Extremely common in modern app design
- **Neumorphic UI** - Requires inner shadows
- **Glass/frosted effects** - Need semi-transparent gradients
- **Custom shadow effects** - Brand-specific shadow colors, multiple shadows
- **Card highlighting** - Gradient borders, glow effects
- **Form field styling** - Gradient backgrounds for inputs

Related issues demonstrate significant community demand:
- [#89563](https://github.com/flutter/flutter/issues/89563) - ElevatedButton filled with gradient
- [#139456](https://github.com/flutter/flutter/issues/139456) - Allow arbitrary foreground and background elements for ButtonStyle
- [#130335](https://github.com/flutter/flutter/issues/130335) - Add Decoration to ButtonStyleButton's styling

---

## Proposal

### High-Level Approach

Introduce a **decoration-aware `ShapeBorder`** system that:

1. **Creates a new `OutlinedBorder` subclass** (`DecorationShapeBorder`) that carries decoration data (gradient, shadows, image, color)
2. **Modifies `Material` widget** to detect and render decoration data from its `shape` property
3. **Maintains full backward compatibility** - existing `ShapeBorder` implementations continue working unchanged
4. **Enables theming** - decoration can be specified in `ThemeData` component themes

### Additional Shape Features

As part of this enhancement, also add support for:
- **Mixed corner types** - Different corner style per corner (rounded, beveled, continuous/superellipse)
- **Per-corner radius** - Different radius for each corner
- Smooth **shape animation/lerping** between different corner configurations

---

## Detailed Proposal

### 1. New Classes in `painting` Library

#### 1.1 `DecorationCornerStyle` Enum

```dart
/// The style of a corner in a [DecorationShapeBorder].
enum DecorationCornerStyle {
  /// Standard circular arc corner (like [RoundedRectangleBorder]).
  rounded,

  /// Cut/chamfered corner (like [BeveledRectangleBorder]).
  beveled,

  /// Smooth continuous curve (like [ContinuousRectangleBorder]).
  continuous,

  /// iOS-style squircle/superellipse (like [RoundedSuperellipseBorder]).
  superellipse,
}
```

#### 1.2 `DecorationCorner` Class

```dart
/// Defines the style and radius of a single corner.
@immutable
class DecorationCorner {
  const DecorationCorner({
    this.style = DecorationCornerStyle.rounded,
    this.radius = Radius.zero,
  });

  /// The visual style of this corner.
  final DecorationCornerStyle style;

  /// The radius of this corner.
  final Radius radius;

  /// Linearly interpolate between two corners.
  static DecorationCorner? lerp(DecorationCorner? a, DecorationCorner? b, double t);

  // ... equality, hashCode, etc.
}
```

#### 1.3 `DecorationCorners` Class

```dart
/// Defines the corners of a [DecorationShapeBorder].
@immutable
class DecorationCorners {
  const DecorationCorners({
    this.topLeft = const DecorationCorner(),
    this.topRight = const DecorationCorner(),
    this.bottomLeft = const DecorationCorner(),
    this.bottomRight = const DecorationCorner(),
  });

  /// Creates corners with the same style and radius.
  const DecorationCorners.all(DecorationCorner corner)
      : topLeft = corner,
        topRight = corner,
        bottomLeft = corner,
        bottomRight = corner;

  /// Creates corners from a [BorderRadius] with a uniform style.
  factory DecorationCorners.fromBorderRadius(
    BorderRadius borderRadius, {
    DecorationCornerStyle style = DecorationCornerStyle.rounded,
  });

  final DecorationCorner topLeft;
  final DecorationCorner topRight;
  final DecorationCorner bottomLeft;
  final DecorationCorner bottomRight;

  /// Linearly interpolate between two corner sets.
  static DecorationCorners? lerp(DecorationCorners? a, DecorationCorners? b, double t);

  // ... equality, hashCode, etc.
}
```

#### 1.4 `DecorationShadow` Class

```dart
/// A shadow specification that supports inner shadows.
@immutable
class DecorationShadow {
  const DecorationShadow({
    this.color = const Color(0x33000000),
    this.offset = Offset.zero,
    this.blurRadius = 0.0,
    this.spreadRadius = 0.0,
    this.inset = false,
  });

  final Color color;
  final Offset offset;
  final double blurRadius;
  final double spreadRadius;

  /// If true, the shadow is drawn inside the shape (inner shadow).
  final bool inset;

  /// Converts to a [BoxShadow] (for outer shadows only).
  BoxShadow toBoxShadow();

  /// Linearly interpolate between two shadows.
  static DecorationShadow? lerp(DecorationShadow? a, DecorationShadow? b, double t);

  /// Linearly interpolate between two shadow lists.
  static List<DecorationShadow> lerpList(
    List<DecorationShadow>? a,
    List<DecorationShadow>? b,
    double t,
  );
}
```

#### 1.5 `DecorationShapeBorder` Class

```dart
/// A [ShapeBorder] that carries decoration data for rendering by [Material].
///
/// When used as the `shape` of a [Material] widget, the decoration properties
/// (gradient, shadows, image) will be rendered by the Material.
///
/// Example:
/// ```dart
/// Card(
///   shape: DecorationShapeBorder(
///     corners: DecorationCorners.all(
///       DecorationCorner(style: DecorationCornerStyle.superellipse, radius: Radius.circular(16)),
///     ),
///     gradient: LinearGradient(colors: [Colors.blue, Colors.purple]),
///     shadows: [
///       DecorationShadow(color: Colors.blue.withOpacity(0.3), blurRadius: 12, offset: Offset(0, 6)),
///     ],
///   ),
///   child: content,
/// )
/// ```
class DecorationShapeBorder extends OutlinedBorder {
  const DecorationShapeBorder({
    this.corners = const DecorationCorners(),
    super.side,
    this.color,
    this.gradient,
    this.shadows,
    this.image,
  }) : assert(color == null || gradient == null,
           'Cannot provide both a color and a gradient');

  /// The corners of this shape.
  final DecorationCorners corners;

  /// The color to fill the shape with.
  ///
  /// If [gradient] is also specified, [gradient] takes precedence.
  final Color? color;

  /// The gradient to fill the shape with.
  final Gradient? gradient;

  /// The shadows to paint around (and optionally inside) the shape.
  final List<DecorationShadow>? shadows;

  /// An image to paint inside the shape.
  final DecorationImage? image;

  /// Whether this border has decoration data that should be rendered.
  bool get hasDecoration => color != null || gradient != null ||
      (shadows != null && shadows!.isNotEmpty) || image != null;

  @override
  Path getOuterPath(Rect rect, {TextDirection? textDirection}) {
    return _DecorationPathBuilder.build(rect, corners);
  }

  @override
  Path getInnerPath(Rect rect, {TextDirection? textDirection}) {
    return _DecorationPathBuilder.build(
      rect.deflate(side.strokeInset),
      corners,
    );
  }

  @override
  void paint(Canvas canvas, Rect rect, {TextDirection? textDirection}) {
    // Only paints the border stroke - fill is handled by Material
    if (side != BorderSide.none) {
      final path = getOuterPath(rect, textDirection: textDirection);
      canvas.drawPath(path, side.toPaint());
    }
  }

  @override
  DecorationShapeBorder copyWith({
    BorderSide? side,
    DecorationCorners? corners,
    Color? color,
    Gradient? gradient,
    List<DecorationShadow>? shadows,
    DecorationImage? image,
  });

  @override
  ShapeBorder? lerpFrom(ShapeBorder? a, double t);

  @override
  ShapeBorder? lerpTo(ShapeBorder? b, double t);

  @override
  ShapeBorder scale(double t);
}
```

### 2. Modifications to `Material` Widget

The `Material` widget (and its internal `_MaterialInterior`) needs to detect when its `shape` is a `DecorationShapeBorder` and render the decoration accordingly.

#### 2.1 Detection Mechanism

Add an interface that `DecorationShapeBorder` implements:

```dart
/// Interface for [ShapeBorder]s that provide decoration data.
abstract class DecorationProvider {
  /// The color to fill the shape with.
  Color? get decorationColor;

  /// The gradient to fill the shape with.
  Gradient? get decorationGradient;

  /// The shadows to paint.
  List<DecorationShadow>? get decorationShadows;

  /// An image to paint inside the shape.
  DecorationImage? get decorationImage;

  /// Whether this provider has any decoration data.
  bool get hasDecoration;
}
```

#### 2.2 Material Widget Changes

In `_MaterialInterior` and related render objects:

```dart
// Pseudocode for Material's build method
Widget build(BuildContext context) {
  final shape = widget.shape;

  // Check if shape provides decoration data
  if (shape is DecorationProvider && shape.hasDecoration) {
    // Use decoration-aware rendering
    return _DecorationMaterial(
      shape: shape,
      decorationProvider: shape as DecorationProvider,
      clipBehavior: widget.clipBehavior,
      child: contents,
    );
  }

  // Existing Material rendering path (backward compatible)
  return _MaterialInterior(
    shape: shape,
    elevation: widget.elevation,
    color: backgroundColor,
    // ... existing implementation
  );
}
```

#### 2.3 Decoration Rendering

The `_DecorationMaterial` widget would:

1. **Paint outer shadows** - Non-inset `DecorationShadow` items
2. **Paint fill** - Using `color` or `gradient`
3. **Paint image** - Clipped to shape path
4. **Paint inner shadows** - Inset `DecorationShadow` items (using path subtraction technique)
5. **Paint border** - Using `side` property
6. **Render child** - With proper clipping

```dart
class _DecorationMaterialPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final rect = Offset.zero & size;
    final path = shape.getOuterPath(rect);

    // 1. Outer shadows
    for (final shadow in shadows.where((s) => !s.inset)) {
      _paintOuterShadow(canvas, path, shadow);
    }

    // 2. Fill (color or gradient)
    if (gradient != null) {
      final paint = Paint()..shader = gradient!.createShader(rect);
      canvas.drawPath(path, paint);
    } else if (color != null) {
      canvas.drawPath(path, Paint()..color = color!);
    }

    // 3. Image (clipped)
    if (image != null) {
      canvas.save();
      canvas.clipPath(path);
      _paintImage(canvas, rect);
      canvas.restore();
    }

    // 4. Inner shadows
    if (shadows.any((s) => s.inset)) {
      canvas.save();
      canvas.clipPath(path);
      for (final shadow in shadows.where((s) => s.inset)) {
        _paintInnerShadow(canvas, rect, path, shadow);
      }
      canvas.restore();
    }
  }

  void _paintInnerShadow(Canvas canvas, Rect rect, Path path, DecorationShadow shadow) {
    // Inner shadow technique: paint shadow on inverted path
    final outerRect = rect.inflate(shadow.blurRadius * 2 + shadow.spreadRadius.abs());
    final outerPath = Path()..addRect(outerRect);
    final innerPath = path.shift(shadow.offset);
    // Shrink for spread (inner shadows shrink with positive spread)
    final shadowPath = Path.combine(PathOperation.difference, outerPath, innerPath);

    final paint = Paint()
      ..color = shadow.color
      ..maskFilter = MaskFilter.blur(BlurStyle.normal, shadow.blurRadius);

    canvas.drawPath(shadowPath, paint);
  }
}
```

### 3. Theme Integration

#### 3.1 Component Themes

The `DecorationShapeBorder` can be used directly in component themes:

```dart
ThemeData(
  cardTheme: CardTheme(
    shape: DecorationShapeBorder(
      corners: DecorationCorners.all(
        DecorationCorner(style: DecorationCornerStyle.superellipse, radius: Radius.circular(16)),
      ),
      gradient: LinearGradient(
        colors: [Colors.blue.shade50, Colors.blue.shade100],
      ),
      shadows: [
        DecorationShadow(
          color: Colors.blue.withOpacity(0.2),
          blurRadius: 20,
          offset: Offset(0, 10),
        ),
      ],
    ),
  ),
  dialogTheme: DialogTheme(
    shape: DecorationShapeBorder(
      corners: DecorationCorners.all(
        DecorationCorner(style: DecorationCornerStyle.rounded, radius: Radius.circular(28)),
      ),
      shadows: [
        DecorationShadow(color: Colors.black26, blurRadius: 24, offset: Offset(0, 8)),
        DecorationShadow(color: Colors.white24, blurRadius: 4, inset: true), // Inner highlight
      ],
    ),
  ),
)
```

### 4. Animation Support

The `DecorationShapeBorder` fully supports animation through standard `lerpFrom`/`lerpTo`:

```dart
@override
ShapeBorder? lerpTo(ShapeBorder? b, double t) {
  if (b is DecorationShapeBorder) {
    return DecorationShapeBorder(
      corners: DecorationCorners.lerp(corners, b.corners, t)!,
      side: BorderSide.lerp(side, b.side, t),
      color: Color.lerp(color, b.color, t),
      gradient: Gradient.lerp(gradient, b.gradient, t),
      shadows: DecorationShadow.lerpList(shadows, b.shadows, t),
      // image lerping could transition opacity
    );
  }
  return super.lerpTo(b, t);
}
```

This enables smooth transitions when:
- Widget state changes (hover, pressed, focus)
- Theme changes
- Explicit animations

---

## API Impact Analysis

### Breaking Changes: None

This proposal introduces no breaking changes:
- Existing `ShapeBorder` implementations continue working unchanged
- `Material` widget maintains backward compatibility
- All existing theme configurations remain valid

### New Public API

**New classes in `painting` library:**
- `DecorationCornerStyle` (enum)
- `DecorationCorner`
- `DecorationCorners`
- `DecorationShadow`
- `DecorationShapeBorder`
- `DecorationProvider` (interface)

### Modified Classes

**`Material` widget:**
- Internal detection of `DecorationProvider` shapes
- New rendering path for decoration-aware shapes

---

## Benefits

1. **Unified Styling API** - Use the same `shape` property for both geometry and decoration
2. **Theme Integration** - Decoration can be specified in `ThemeData` component themes
3. **Declarative** - No builder functions required, just properties
4. **Backward Compatible** - Existing code continues working
5. **Full Material Behavior** - Ink splash, focus, accessibility all preserved
6. **Animation Support** - Smooth transitions between states and themes
7. **Mixed Corner Types** - Per-corner style and radius
8. **Inner Shadows** - Enable neumorphic and modern UI patterns

---

## Alternatives Considered

### 1. `backgroundBuilder` Pattern (Current Approach for Buttons)

Flutter 3.22 added `backgroundBuilder` to `ButtonStyle`. This could be extended to other components.

**Pros:**
- Already implemented for buttons
- Maximum flexibility

**Cons:**
- Requires builder function (not declarative)
- Cannot be used in static theme definitions
- Must be implemented per-component
- Different API pattern than `shape` property

### 2. New `decoration` Property on Material Components

Add a separate `decoration` property to `Card`, `Dialog`, etc.

**Cons:**
- Adds another property (API bloat)
- Unclear interaction with `shape` and `color` properties
- Would need separate property in each component theme

### 3. `ShapeDecoration` Integration

Use existing `ShapeDecoration` class more directly.

**Cons:**
- `ShapeDecoration` is a `Decoration`, not a `ShapeBorder`
- Cannot be assigned to `shape` properties
- Would require different API

---

## Implementation Phases

### Phase 1: Core Classes
- Implement `DecorationCorner`, `DecorationCorners`, `DecorationShadow`
- Implement `DecorationShapeBorder`
- Add `DecorationProvider` interface
- Unit tests for lerping and path generation

### Phase 2: Material Integration
- Modify `Material` widget to detect and render `DecorationProvider` shapes
- Implement decoration painting (gradients, shadows, images)
- Inner shadow rendering
- Integration tests

### Phase 3: Documentation and Examples
- API documentation
- Migration guide
- Example app demonstrating usage with various components
- Performance benchmarks

---

## Related Issues

- [#89563](https://github.com/flutter/flutter/issues/89563) - ElevatedButton filled with gradient
- [#139456](https://github.com/flutter/flutter/issues/139456) - Allow arbitrary foreground and background elements for ButtonStyle
- [#130335](https://github.com/flutter/flutter/issues/130335) - Add Decoration to ButtonStyleButton's styling
- [#19147](https://github.com/flutter/flutter/issues/19147) - shape: RoundedRectangleBorder + Border.all -> Not Round
- [#104201](https://github.com/flutter/flutter/issues/104201) - Request: Add a way to round inside shape border

---

## Summary

This proposal enables Flutter developers to use rich decoration styling (gradients, shadows, images) on Material components through the existing `shape` property, with full theme integration, animation support, and backward compatibility. It addresses a significant gap in Flutter's styling capabilities that has generated numerous related feature requests and workarounds in the community.