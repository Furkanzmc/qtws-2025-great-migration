---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: The Great Migration
info: |
  QtWS 2025: The Great Migration
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
# seoMeta:
#  ogImage: https://cover.sli.dev
---

<style>
.slidev-code {
    height: 90%;
    overflow-y: scroll;
}
</style>

# The Great Migration: Transferring Millions of Data Objects to QML with Speed and Comfort

> Furkan Uzumcu
>
> ·[Autodesk](https://www.autodesk.com/products/alias-products/overview)
>
> [GitHub/Furkanzmc](https://github.com/Furkanzmc)
>
> [zmc.space](https://zmc.space)
>
> [furkan.uzumcu@autodesk.com](mailto:furkan.uzumcu@autodesk.com)

---

# What is Alias?

<Youtube autoplay id="Q3INqSNbmZU" width="100%" height="100%" />

---

# Prologue

- Migrating legacy GUI to Qt/QML
- Pure QML/Qt Quick GUI implementation
- Lots and lots of data to transfer to GUI and back
- Extensive use of an internal data structure that we cannot change

---

# Data Representation in the Core

- A `Symbol` is like `QProperty`, a `SymbolTable` is like a `QVariantMap`.

```cpp
SymbolTable data{
    {"degrees", IntSymbol{}},
    {"spans", IntSymbol{}},
    {"value", StringSymbol{}},
    {"calculate", FunctionSymbol{[]() {}}},
};
```

- Data for UI is created procedurally in thousand different places.
```cpp
void create() {
  ControlBox box;
  box.addInt(IntegerSymbol{}, "Degrees");
  box.addInt(IntegerSymbol{}, "Spans");
  box.addSring(StringSymbol{}, "Value");
  box.addSring(FunctionSymbol{[]() {}}, "calculate");
}
```

---

# Initial Attempt

```mermaid
classDiagram
CoreEditor *-- UIEditor
UIEditor : ControlBase[] controls
UIEditor : QQmlPropertyMap dynamicProperties
UIEditor : open()
UIEditor : close()
UIEditor *-- Editor1
UIEditor *-- Editor2
UIEditor *-- EditorN

ControlBase *-- ControlInt
ControlBase *-- ControlString
ControlBase *-- ControlVector3D
ControlBase *-- ControlFloat
ControlBase *-- ControlX
```

---

```cpp
class Core::MyEditor {
  virtual void open() = 0;
  virtual void close() = 0;
  std::string getData() const;
  void setData(const std::string &data);
};
```

<br>
<br>

<v-click>

```cpp
class UI::MyEditor : public Core::MyEditor {
  void open() final {}
  void close() final{};
  QString getQData() const;
  void setQData(const QString &data);
  QVector<ControlBase*> controls() const;
  QQmlPropertyMap dynamicProperties() const;
};
```

</v-click>

<!--
- When we needed to expose data to UI, we needed to have a class that inherits from the core and
  exposes it.
- We used `QQmlPropertyMap` as well.
- `QQmlPropertyMap`
-->

---

```cpp
class awQuick::ControlColor : public ControlBase {
  Q_PROPERTY(QColor value READ getValue WRITE setValue NOTIFY valueChanged)

public:
  ControlColor(TripleSymbol *symbol, const char *title);
  QColor getValue() const;
  void setValue(QColor val);

signals:
  void valueChanged();

private:
  TripleSymbol *m_sym{nullptr};
  QColor m_color{};
};
```

<!--
- We already had thousands and thousands lines of code that uses this pattern.
- There are editors with thousands, sometimes millions of these calls.
-->

---

# Model/View For UI Controls

```qml
Editor {
    property var editorData

    Repeater {
        model: editorData.controls
        delegate: Loader {
            id: ld

            // Information is lost...
            property var modelData

            // Information is still lost...
            property ControlBase modelData

            // One of many possibilities...
            ColorEditDelegate {
                color: modelData.value
            }
        }
    }
}
```

---

# Getting Additional Data from an Editor

```qml
Editor {
    property var editorData

    customProperty: editorData.dynamicProperties.customData
    secondTitle: editorData.dynamicProperties.secondTitle
}
```

---

What happens when we want to call a function from our data table?

<v-click>

```cpp
Q_INVOKABLE void calculate() { m_functionSymbol.invoke(); }

// Or we cal into some virtual method that calls the underlying function symbol
Q_INVOKABLE void calculate() { calculateInternal(); }
```

</v-click>

<v-click>

What happens when you want to just arbitrarily expose function symbols?..

</v-click>

<v-click>

```cpp
Q_INVOKABLE void notFun1() {}
Q_INVOKABLE void notFun2() {}
Q_INVOKABLE void notFun3() {}
Q_INVOKABLE void notFun4() {}
```

</v-click>

<!--
- We want to do as much work as possible in the core, all the logic stays there. Including opening
  dialogs, updating properties etc.
-->

---

# Cons

- It's hard to use static types
> We have to rely on `var` property type
- Each change requires a re-build
- Lots of virtual interfaces...
- Code bloat
- Function calls has to be converted to `Q_INVOKABLE` methods

<br>

<v-click>

# Pros

- It's C++?..

</v-click>

---

# Realization

![this-sucks](./assets/this-sucks.gif)

---

# Other Realizations

- We have way too much boilerplate code to expose data
- Virtual interfaces create a dependency that we don't like
- Our data is like JSON
- These data types are generic, what they represent just changes based on context
- The symbols we add to the table might be used for other purposes, so we need more freedom over
  how much we expose
- We want to control all the data flow from a single point

---

```cpp
SymbolTable data{
    {"degrees", IntSymbol{32}},
    {"spans", IntSymbol{-1}},
    {"value", StringSymbol{"Timmy!"}},
    {"greet", FunctionSymbol{[]() { std::clog << "Hello, Qt Athens!\n"; }}},
};
```

<br>

```qml
SymbolTableModel {
    property int degrees
    property int spans
    property string value
    property var greet () => {}
    // Will be possible at some point... QTBUG-XXXXX
    property function greet () => {}

    Component.onCompleted: {
        console.log(degrees) // Prints "32"
        console.log(spans) // Prints "-1"
        console.log(value) // Prints "Timmy!"
        greet() // Prints "Hello, Qt Athens!"
    }
}
```

---

More complex cases...

<SlidevVideo autoplay controls>
    <source src="/assets/alias-editors.mp4" type="video/mp4">
</SlidevVideo>

---

Here's the data we use to draw those shader balls.

```qml
SymbolTableModel {
    property int width
    property int height
    property bool viewEnabled
    property SymbolTable drawLayers
    property SymbolTableList items
    property SymbolTable mouse
}
```

---

```qml
SymbolTableModel {
    property string title: ""
    property string type: ""
    property bool visible: true
    property bool enabled: true
}
```

<v-click>

```qml
SymbolTableModel {
    property string title: ""
    property string type: ""
}
```

</v-click>

<v-click>

```qml
SymbolTableModel {
    property string title: ""
    property string type: ""
    property bool visible: true
    property bool enabled: true
}
```

</v-click>

<v-click>

We just expose what we need, no extra code needed to change the data here.

</v-click>

---

Ability to play with syntax to make exposing data more ergonomic.

```qml
SymbolTableModel {
    property SymbolTableModel view: SymbolTableModel {
        property int width
        property int height
        property SymbolTable drawLayers
    }
}
```

---

Ability to customize the underlying mechanics of how data is exposed.

```qml
SymbolTableModel {
    // This will create our special `ControlTableList` type internally and append `SymbolTable` types
    // to it.
    property ControlTableList controls
}
```

We use `ControlTableList` to dynamically filter our model based on some conditions.

---

# How Does It Work?

```mermaid
graph LR;
    A[Symbol is Updated] --> B[SymbolTableModel is Notified];
    B --> C;
    C[QMetaProperty is Updated] --> D[QML Reacts to Changes];
    D --> E;
    E[QML Updates Property] --> B;
    B --> A;

```

---

```cpp
class SymbolTableModel : public QObject, public QQmlParserStatus {
  explicit SymbolTableModel(QObject *parent = nullptr);

  void componentComplete() override final;

  SymbolTableProxy *table() const;
  bool initialized() const;

private:
  SymbolBindingMap m_symbolBindings{};
  TableBindingMap m_tableBindings{};
  SignalBindingMap m_signalBindings{};

  std::stack<SymbolPropertyBinding *> m_invalidSymbolBindings{};
  std::stack<SymbolTablePropertyBinding *> m_invalidTableBindings{};
  std::stack<SymbolSignalBinding *> m_invalidSignalBindings;{}

  std::unique_ptr<PropertyCache> m_propertyCache{};
  std::unique_ptr<MethodCache> m_methodCache{};

private:
  void setTable(SymbolTable_H value);
  void initializeProperties();
  void deinitializeProperties();
  void disconnectProperties();
};
```

---

```cpp
class SymbolPropertyBindingBase {};

class SymbolPropertyBinding : public SymbolPropertyBindingBase {};

class SymbolTablePropertyBinding : public SymbolPropertyBindingBase {};

class awQuick::SymbolSignalBinding : public SymbolPropertyBindingBase {};

class awQuick::QMLFunctionSymbolExecutor : public QObject {
public:
  QMLFunctionSymbolExecutor(const FunctionSymbol *funcSymbol, QObject &parent);

  Q_INVOKABLE QVariant call(const QVariantList &args) const;

private:
  const TuiFunctionSymbol *m_funcSymbol{nullptr};
};
```

---

```cpp {all|1-1|2-3|4-12|13-18|all}
if (symbolType == TuiSymbol::Type::UI_FUNCTION_SYMBOL) {
  QJSEngine *engine = qjsEngine(obj.parent());
  assert(engine);

  QJSValue function =
      engine->evaluate("(function(obj) { return (function() { "
                       "let args = [];"
                       "for (let i = 0; i < arguments.length; i++) {"
                       " args.push(arguments[i])"
                       "};"
                       "return this.call(args); }).bind(obj) })");
  assert(function.isCallable());

  QJSValue boundFunction = function.call(
      QJSValueList{engine->toScriptValue(new QMLFunctionSymbolExecutor{
          static_cast<const TuiFunctionSymbol *>(&symbol),
          const_cast<QObject &>(obj)})});
  assert(boundFunction.isCallable());

  return QVariant::fromValue<QJSValue>(boundFunction);
}
```

---

```cpp
// Also supports return types.
QVariant QMLFunctionSymbolExecutor::call(const QVariantList &args) const {
  assert(m_funcSymbol);
  FunctionSymbol::ArgList argList;

  auto *base = qobject_cast<SymbolPropertyBindingBase *>(parent());

  if (!args.isEmpty()) {
    for (const auto &arg : args) {
      argList.push_back(convertVariant(arg));
    }
  }

  return argToQVariant(m_funcSymbol->invoke(argList));
}
```

---

# Where do we go from here?

- No performance issues yet, but we'll investigate to make it
- Look into QML compiler extension to see how this would work
- Make introspection and debugging easier

---

# Thank You!
