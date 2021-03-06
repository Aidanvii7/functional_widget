[![Build Status](https://travis-ci.org/rrousselGit/functional_widget.svg?branch=master)](https://travis-ci.org/rrousselGit/functional_widget)
[![pub package](https://img.shields.io/pub/v/functional_widget.svg)](https://pub.dartlang.org/packages/functional_widget) [![pub package](https://img.shields.io/badge/Awesome-Flutter-blue.svg?longCache=true&style=flat-square)](https://github.com/Solido/awesome-flutter) [![codecov](https://codecov.io/gh/rrousselGit/functional_widget/branch/master/graph/badge.svg)](https://codecov.io/gh/rrousselGit/functional_widget)

Widgets are cool. But classes are quite verbose:

```dart
class Foo extends StatelessWidget {
  final int value;
  final int value2;

  const Foo({Key key, this.value, this.value2}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Text('$value $value2');
  }
}
```

So much code for something that could be done much better using a plain function:

```dart
Widget foo(BuildContext context, { int value, int value2 }) {
  return Text('$value $value2');
}
```

The problem is, using functions instead of classes is not recommended:

- https://stackoverflow.com/questions/53234825/what-is-the-difference-between-functions-and-classes-to-create-widgets/53234826#53234826
- https://github.com/flutter/flutter/issues/19269

... Or is it?

---

_functional_widgets_, is an attempt to solve this issue, using a code generator.

Simply write your widget as a function, decorate it with a `@widget`, and then this library will generate a class for you to use.

## Example

You write:

```dart
@widget
Widget foo(BuildContext context, int value) {
  return Text('$value');
}
```

It generates:

```dart
class Foo extends StatelessWidget {
  final int value;

  const Foo(this.value, {Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return foo(context, value);
  }
}
```

And then you use it:

```dart
runApp(
    Foo(42)
);
```

## How to use

### Install (builder)

There are a few separate packages you need to install:

- `functional_widget_annotation`, a package containing decorators. You must install it as `dependencies`.
- `functional_widget`, a code-generator that uses the decorators from the previous packages to generate your widget.
Install it inside `builders`, a Flutter specific field on your `pubspec.yaml` made for code-generators.

Your `pubspec.yaml` should looks like:

```yaml
dependencies:
  functional_widget_annotation: ^0.5.0

builders:
  functional_widget: ^0.6.0
```

Then indicate that you are using generated file using Dart's `part` keyword. For example, **my_file.dart** might look like this
```dart
import 'package:flutter/material.dart';

part 'my_file.g.dart';

@widget
Widget foo(BuildContext context, int value) {
  return Text('$value');
}

// now you can use Foo(42)
```

That's it! Flutter will automatically run the code generator when executing `flutter build`, `flutter run` or similar.

### Install (build_runner)

If your version of Flutter is too old, the previous installation method may not work.
In that case it is possible to work with `functional_widget` by using `build_runner` package.

First add the following to your `pubspec.yaml`:

```yaml
dependencies:
  functional_widget_annotation: ^0.5.0

dev_dependencies:
  functional_widget: ^0.6.0
  build_runner: ^1.3.1
```

Then to run the generator, you must use `build_runner`:

```sh
flutter pub pub run build_runner watch
```

This will watch your source folder and run the code-generator whenever something changes.

## Customize the output

It is possible to customize the output of the generator by using different decorators or configuring default values in `build.yaml` file.

`build.yaml` change the default behavior of a configuration.

```yaml
# build.yaml
targets:
  $default:
    builders:
      functional_widget:
        options:
          # Default values:
          debugFillProperties: false
          widgetType: stateless # or 'hook'
          equality: none # or 'identical'/'equal'
```

`FunctionalWidget` decorator will override the default behavior for one specific widget.

```dart
@FunctionalWidget(
  debugFillProperties: true,
  widgetType: FunctionalWidgetType.hook,
  equality: FunctionalWidgetEquality.identical,
)
Widget foo() => Container();
```

### debugFillProperties override

Widgets can be override `debugFillProperties` to display custom fields on the widget inspector. `functional_widget` offer to generate these bits for your, by enabling `debugFillProperties` option.

For this to work, it is required to add the following import:

```dart
import 'package:flutter/foundation.dart';
```

Example:

(You write)

```dart
import 'package:flutter/foundation.dart';

@widget
Widget example(int foo, String bar) => Container();
```

(It generates)

```dart
class Example extends StatelessWidget {
  const Example(this.foo, this.bar, {Key key}) : super(key: key);

  final int foo;

  final String bar;

  @override
  Widget build(BuildContext _context) => example(foo, bar);
  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(IntProperty('foo', foo));
    properties.add(StringProperty('bar', bar));
  }
}
```

### Generate different type of widgets

By default, the generated widget is a `StatelessWidget`.

It is possible to generate a `HookWidget` instead (from https://github.com/rrousselGit/flutter_hooks)

There are a few ways to do so:

- Through `build.yaml`:

```yaml
# build.yaml
targets:
  $default:
    builders:
      functional_widget:
        options:
          widgetType: hook
```

- With `@FunctionalWidget` decorator:

```dart
@FunctionalWidget(widgetType: FunctionalWidgetType.hook)
Widget example(int foo, String bar) => Container();
```

- With the shorthand `@hwidget` decorator:

```dart
@hwidget
Widget example(int foo, String bar) => Container();
```

```dart
class Example extends HookWidget {
  const Example(this.foo, this.bar, {Key key}) : super(key: key);

  final int foo;

  final String bar;

  @override
  Widget build(BuildContext _context) => example(foo, bar);
}
```

---

In any cases, `flutter_hooks` must be added as a separate dependency in the `pubspec.yaml`

```yaml
dependencies:
  flutter_hooks: # some version number
```

### operator== override

It can be interesting for `Widget` to override `operator==` for performance optimizations.

`functional_widget` optionally allows overriding both `operator==` and `hashCode` based on the field used.

There are two different configurations:

- `none` (default): Don't override anything
- `identical`, overrides `hashCode` and `operator==` with the latter being implemented using `identical` to compare fields.
- `equal`, similar to `identical`, but using `==` to compare fields.

It can be configured both through `build.yaml`:

```yaml
# build.yaml
targets:
  $default:
    builders:
      functional_widget:
        options:
          equility: identical
```

or using `@FunctionalWidget` decorator:

```dart
@FunctionalWidget(equality: FunctionalWidgetEquality.identical)
Widget example(int foo, String bar) => Container();
```

### All the potential function prototypes

_functional_widget_ will inject widget specific parameters if you ask for them.
You can potentially write any of the following:

```dart
Widget foo();
Widget foo(BuildContext context);
Widget foo(Key key);
Widget foo(BuildContext context, Key key);
Widget foo(Key key, BuildContext context);
```

You can then add however many arguments you like **after** the previously defined arguments. They will then be added to the class constructor and as a widget field:

- positional

```dart
@widget
Widget foo(int value) => Text(value.toString());

// USAGE

Foo(42);
```

- named:

```dart
@widget
Widget foo({int value}) => Text(value.toString());

// USAGE

Foo(value: 42);
```

- A bit of everything:

```dart
@widget
Widget foo(BuildContext context, int value, { int value2 }) {
  return Text('$value $value2');
}

// USAGE

Foo(42, value2: 24);
```
