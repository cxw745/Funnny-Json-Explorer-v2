# Funny JSON Explorer设计文档

# 任务介绍

> ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/20240615170759.png)​

# 设计文档

## 类图与说明

使用[PlantUML Web Server](https://www.plantuml.com/plantuml/uml/SyfFKj2rKt3CoKnELR1Io4ZDoSa70000) 绘制uml类图，绘制代码位于`class_pic.puml`​。

* ​`fje.py`​: 负责加载JSON文件和图标文件以及解析命令行参数。
* ​`iterator.py`​: 定义用于遍历树结构的迭代器。
* ​`strategy.py`​: 定义可视化策略的抽象类以及具体策略的实现类。
* ​`context.py`​: 定义上下文类，用于设置和使用具体的可视化策略。
* ​`builder.py`​: 定义节点建造者的抽象类以及具体的构建器实现类，实现树构建指挥类，用于使用构建器来创建树结构。
* ​`node.py`​: 定义节点的抽象类以及具体的节点实现类。
* ​`example.json`​: 测试可视化结果的代码。
* ​`icons.json`​: 图标族的配置文件。

​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/20240615170812.png)​

使用**迭代器**和**策略模式**来实现FJE，迭代器模式将用于遍历树结构，策略模式将用于选择和应用不同的可视化风格。

* **迭代器模式**：`TreeIterator`​类用于遍历树结构，`VisualizerContext`​类中使用了这个迭代器来遍历树节点。
* **策略模式**：定义了`VisualizerStrategy`​接口以及三个具体策略类(`TreeVisualizer`​、`RectangleVisualizer`​和`NewVisualizer`​)，通过`VisualizerContext`​类使用不同的策略来实现不同的可视化效果。

### 迭代器模式

**迭代器模式**用于遍历集合对象的元素，而不需要暴露其底层表示。

在代码中，迭代器模式通过`TreeIterator`​类实现，`TreeIterator`​通过栈结构深度优先遍历树。`__iter__`​方法使`TreeIterator`​类成为一个可迭代对象，`__next__`​方法则实现了迭代器协议。

```python
# iterator.py
class TreeIterator:
    def __init__(self, root):
        self.stack = [(root, 0, False)]  # Node, level, is_last_child

    def __iter__(self):
        return self

    def __next__(self):
        if not self.stack:
            raise StopIteration

        node, level, is_last_child = self.stack.pop()
        if isinstance(node, CompositeNode):
            for i, child in enumerate(reversed(node.children)):
                self.stack.append((child, level + 1, i == 0))

        return node, level, is_last_child
```

### 策略模式

**策略模式**定义了一系列算法，并将每个算法封装起来，使它们可以相互替换。策略模式使得算法可独立于使用它的客户而变化。

在代码中，策略模式通过以下类来实现，这里定义了三个具体策略类`TreeVisualizer`​、`RectangleVisualizer`​和`NewVisualizer`​，它们都实现了`VisualizerStrategy`​接口。

```python
# strategy.py
from abc import ABC, abstractmethod


class VisualizerStrategy(ABC):
    def __init__(self, icons):
        self.icons = icons
    @abstractmethod
    def visualize(self, node, level=0, is_last_child=False):
        pass

    @abstractmethod
    def beautify(self, result):
        pass


class TreeVisualizer(VisualizerStrategy):
    ...


class RectangleVisualizer(VisualizerStrategy):
    ...


class NewVisualizer(VisualizerStrategy):
    ...
```

### 使用模式

使用迭代器模式和策略模式的代码段如下。

迭代器模式中，`TreeIterator`​被用于遍历树节点，逐一将节点交给策略对象进行处理。

策略模式中，`VisualizerContext`​持有一个`VisualizerStrategy`​对象，并在遍历树节点时调用`VisualizerStrategy`​对象的`visualize`​方法。具体策略可以在运行时进行替换，从而实现不同的可视化效果。

```python
# context.py
from iterator import TreeIterator
from strategy import VisualizerStrategy

class VisualizerContext:
    def __init__(self, strategy: VisualizerStrategy):
        self._strategy = strategy

    def set_strategy(self, strategy: VisualizerStrategy):
        self._strategy = strategy

    def visualize(self, node):
        iterator = TreeIterator(node)
        result = ""
        for node, level, is_last_child in iterator:
            result += self._strategy.visualize(node, level, is_last_child)
        return self._strategy.beautify(result)
```

# 结果展示

## 实现要求

1. 不改变现有代码，只需添加新的抽象工厂，即可添加新的风格。

    在`strategy.py`​中，通过继承类**VisualizerStrategy**并重写**visualize**方法即可添加新的风格。

    示例如下，创建一个新的风格NewVisulizer，并在具体可视化工厂中添加创建实际的工厂选项，在执行脚本时，参数`-s`​后跟新添加的可视化风格名称即可使用新的可视化风格。

    ```python
    class NewVisualizer(VisualizerStrategy):
        def visualize(self, node, level=0, is_last_child=False):
            result = "This is new style!"
            indent = " " * level * 2
            if node.is_leaf():
                result += f"{indent}{self.icons['leaf']}{node.name}: {node.value}\n"
            else:
                result += f"{indent}{self.icons['composite']}{node.name}\n"
            return result

        def beautify(self, result):
            return result
    ```

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/20240612110822.png)​
2. 通过配置文件，可添加新的图标族。

    在`icons.json`​中添加新的图标，在执行脚本时，参数`-i`​后跟新添加的图标组名称即可使用新的图标。

    示例如下，添加了math图标组并使用。

    ```json
    {
      "poker": {
        "composite": "♢",
        "leaf": "♤"
      },
      "star": {
        "composite": "✪",
        "leaf": "★"
      },
      "math": {
        "composite": "@",
        "leaf": "#"
      }
    }
    ```

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/20240612110937.png)​

## 运行截图

* Tree

  * poker

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/20240612112603.png)​
  * star

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/image-20240612103138-crp5wcq.png)​
* Rectangle

  * poker

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/image-20240612103148-fredwyf.png)​
  * star

    ​![image](https://cdn.jsdelivr.net/gh/cxw745/ImgBed/image-20240612103153-57yh77x.png)​

‍
