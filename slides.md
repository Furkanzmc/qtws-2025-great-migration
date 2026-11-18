---
# You can also start simply with 'default'
theme: dracula
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
layout: image
image: ./assets/wave-cartman.gif
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

<!--
- The declaration of a `SymbolTableModel` is also a contract that designers can easily see without
  getting into C++.
-->

---

# Prologue

- Migrating legacy GUI to Qt/QML
- Pure QML/Qt Quick GUI implementation
- Lots and lots of data to transfer to GUI and back
- Extensive use of an internal data structure that we cannot change
> Refer to [QtWS 2023: Porting a Large Scale Non-Qt Legacy CAD Application to Qt/QML: The Good, The
> Bad, and The Ugly](https://www.youtube.com/watch?v=7Omr6hlxEhY) for details.

<!--
- The project took 5 years to complete.
- Refer to my other talk for details.
-->

---

# What are the problems we are trying to solve?

- Code bloat
- Developer productivity
- Dependencies
> Ideally, we want to be able to run GUI without the underlying system
- System inflexibility
- Developer focused GUI development

<!--
- The goal is to make sure that we are not constrained by our architecture, and UX can go crazy with
  whatever they want to do.
-->

---

# What is the dream?

- Maximum freedom for designing new user experiences
- Make our system more configurable
- Remove developers as a bottleneck for UX improvements

---
layout: center
---

# What this looks like in practice...

---
layout: center
---

<div class="grid grid-cols-2 gap-2">
  <div>
    <img src="./assets/custom-puck.gif"/>
  </div>
  <div>
    <img src="./assets/alias-layerbar-2026.gif"/>
  </div>
</div>

<!--
- Uses the same data source as our menus in a different way
-->

---
layout: center
---

<div class="grid grid-cols-2 gap-2">
  <div>
    <img src="./assets/transform-cv.png"/>
  </div>
  <div>
    <SlidevVideo autoplay controls>
        <source src="/assets/alias-editors.mp4" type="video/mp4">
    </SlidevVideo>
  </div>
</div>

---

<SlidevVideo autoplay controls>
    <source src="/assets/alias-classic-vs-modern.mp4" type="video/mp4">
</SlidevVideo>

---
layout: image
image: ./assets/come-on-loosen-up.gif
backgroundSize: 40%
---

<!--
- In order to achieve those results, we need to loosen up our system.
-->

---
layout: center
---

# How do we represent data?

---

# Data Representation in the Core

- A `Symbol` is like `QProperty`, a `SymbolTable` is like a `QVariantMap`.

```cpp
// NOTE: Crude representation
SymbolTable data{
    {"degrees", IntSymbol{}},
    {"spans", IntSymbol{}},
    {"value", StringSymbol{}},
    {"calculate", FunctionSymbol{[]() {}}},
};
```

<v-click>

- Data for UI is created procedurally in lots of different places.
```cpp
void create() {
  // NOTE: Also code we cannot change.
  ControlBox box;
  box.addInt(IntegerSymbol{}, "Degrees"); // Represents an integer input field
  box.addInt(IntegerSymbol{}, "Spans");
  box.addSring(StringSymbol{}, "Value"); // Represents a string input field
  box.addSring(FunctionSymbol{[]() {}}, "calculate");
  box.open();
}
```

</v-click>

<!--
- Our goal is to somehow get all this data into the UI.
-->

---

# Challenge

- Get all this information to the GUI

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
layout: center
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

```cpp
class UI::MyEditor : public Core::MyEditor {
  void open() final {}
  void close() final {};
  QString getQData() const;
  void setQData(const QString &data);
  QVector<ControlBase *> controls() const;
  // Used whenever we had non-standard editor or data to expose.
  QQmlPropertyMap dynamicProperties() const;
};
```

<!--
- When we needed to expose data to UI, we needed to have a class that inherits from the core and
  exposes it.
- We used `QQmlPropertyMap` as well.
-->

---
layout: center
---

```cpp
class ControlColor : public ControlBase {
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

---

# Model/View For UI Controls

```qml
Editor {
    property var editorData

    Repeater {
        model: editorData.controls
        delegate: Loader {
            // Information is lost...
            property var modelData

            // Information is still lost...
            property ControlBase modelData

            // One of many possibilities...
            sourceComponent: ColorEditDelegate {
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

    // `dynamicProperties` is a `QQmlPropertyMap`
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

<v-click>

What happens when requirements change and we need to expose different things?

</v-click>

<v-click>

```cpp
class ControlColor : public ControlBase {
  // ...
  Q_PROPERTY(float newData READ getNewData WRITE setNewData NOTIFY newDataChanged)
};
```

And recompile...

</v-click>

<!--
- We want to do as much work as possible in the core, all the logic stays there. Including opening
  dialogs, updating properties etc.
-->

---

# Cons

- It's hard to use static types
> We have to rely on `var` property type
- Each change requires a re-build in multiple units
- Lots of virtual interfaces...
- Code bloat
- Function calls has to be converted to `Q_INVOKABLE` methods

<br>

<v-click>

# Pros

- It's C++?..

</v-click>

---

# Realization...

---
layout: center
---

![this-sucks](./assets/this-sucks.gif)

---

# Other Realizations

- Our GUI data is JSON-like
- Too much boilerplate code to expose data
- Virtual interfaces create a dependency that we don't like
- These data types are generic, what they represent just changes based on context
- We want to control all the data flow from a single point

<!--
- Ideally, we want to be able to run the GUI without any core involvement at all.
- So that we can run Alias faster with mock data.
-->

---

# Enter `QMetaObject` "Abuse"

```cpp
// What our data looks like...
SymbolTable data{
    {"spans", IntSymbol{-1}},
    {"faceColor", TuiTripleSymbol{255, 132, 132}},
    {"value", StringSymbol{"Timmy!"}},
    {"greet", FunctionSymbol{[]() { std::clog << "Hello all!\n"; }}},
    {"menu", SymbolTable{/* Data that represents menu items */}},
};
```

<v-click>

````md magic-move
```qml
// We have all the types we need in the QML type system
SymbolTableModel {
    property int number
    property string str
    property color colorType
    property point position
    // And also the ability for custom value types and object types.
    property myCustomPoint customPosition
    property MyCustomObject customObject
}
```
```qml
SymbolTableModel {
    property color faceColor
    property int spans
    property string value
    property var greet: () => {}
    property function greet: () => {} // Will be possible at some point... QTBUG-104220
    property Menu menu

    Component.onCompleted: {
        console.log(degrees) // Prints "32"
        console.log(spans) // Prints "-1"
        console.log(value) // Prints "Timmy!"
        greet() // Prints "Hello all!"
    }
}
```
````

</v-click>

---

Refresher...

<SlidevVideo autoplay controls>
    <source src="/assets/alias-editors.mp4" type="video/mp4">
</SlidevVideo>

---

Here's the data we use to draw those shader balls.

````md magic-move
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
```qml
// We can just declare what we want to use.
// Maybe used to visualize a disabled state in another control
SymbolTableModel {
    property int width
    property int height
    property bool viewEnabled
}
```
```qml
SymbolTableModel {
    // Or add more that didn't exist before
    property string title
    property int width
    property int height
    property bool viewEnabled
    property SymbolTable drawLayers
    property SymbolTableList items
    property SymbolTable mouse
}
```
````

---

**Supports inheritance**

```qml

// BaseControl.qml
SymbolTableModel {
    property string title: ""
    property string type: ""
}

// CustomControl.qml
BaseControl {
    property int counter: 0
}
```

---

**Supports all built-int types and more...**

```qml

// BaseControl.qml
SymbolTableModel {
    // Property declarations for all QML types
    property int number
    property real floatingPoint
    property string str
    property size size
    property rect rectangle
    property point position
    // ...
}
```

---

**Ability to play with syntax to make exposing data more ergonomic**

````md magic-move
```qml
SymbolTableModel {
    id: model

    property SymbolTable view
}

SymbolTableModel {
    property int width
    property int height
    property SymbolTable drawLayers

    table: model.view
}
```
```qml
SymbolTableModel {
    property SymbolTableModel view: SymbolTableModel {
        property int width
        property int height
        property SymbolTable drawLayers
    }
}
```
````

<v-click>

Ability to customize the underlying mechanics of how data is exposed.

```qml
SymbolTableModel {
    // This will create our special `ControlTableList` type internally and append `SymbolTable` types
    // to it.
    property ControlTableList controls
}
```

We use `ControlTableList` to dynamically filter our model based on some conditions.

</v-click>

---

The way the underlying data is expressed is fully customizable.

```qml
SymbolTableModel {
    property Person manager: Person {
        property string name
        // And other related properties...
    }
}
```

---
layout: center
---

# How Does It Work?

---

```qml {1-3,10|1-6,9,10|all}
SymbolTableModel {
    property var calculateThings: () => {} // Syntax sugar
    property var getSomeValue

    Component.onCompleted: {
        calculateThings()
        // Functions can also have return values!
        const value = getSomeValue()
    }
}
```

---

```mermaid
graph LR;
    A[Symbol is Updated] --> B[SymbolTableModel is Notified];
    B --> C;
    C[QMetaProperty is Updated] --> D[QML Reacts to Changes];
```


```mermaid
graph LR;
    E[QML Updates Property] --> B[SymbolTableModel is Notified];
    B --> A[Symbol is Updated];
```

<v-click>

````md magic-move
```cpp
// SomeWhereOverTheRainbow.cpp
symbol.notify();
```
```cpp
// SomeWhereOverTheRainbow.cpp
symbol.notify();

// SymbolTableModel.cpp
symbol.sigModified.connect(this, [](...) {
    // The change is handled and the associated `QProperty` is updated.
})
```
````

</v-click>

<v-click>

```qml
SymbolTableModel {
    property int symbol

    onSymbolChanged: {
        console.log("Property value changed")
    }
}
```

</v-click>

---

<SlidevVideo autoplay controls>
    <source src="/assets/object-lister-information-window.mov" type="video/mp4">
</SlidevVideo>

---

# Where do we go from here?

- No performance issues yet, we are on the look out for potential bottlenecks
> For example:
> - Executing JS is expensive, so we cache what we can for each `QJSEngine`
- Look into QML compiler extension to see how this would work
- Make introspection and debugging easier

---
layout: image-left
image: ./assets/adios.gif
---

# Thank you!
