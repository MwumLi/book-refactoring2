# 第 11 章 重构 API

模块和函数是软件的骨肉，而 API 则是将骨肉连接起来的关节。易于理解和使用的 API 非常重要，但同时也很难获得。随着对软件理解的加深，我会学到如何改进 API，这时我便需要对 API 进行重构。

好的 API 会把更新数据的函数与只是读取数据的函数清晰分开。如果我看到这两类操作被混在一起，就会用将查询函数和修改函数分离（306）将它们分开。如果两个函数的功能非常相似、只有一些数值不同，我可以用函数参数化（310）将其统一。但有些参数其实只是一个标记，根据这个标记的不同，函数会有截然不同的行为，此时最好用移除标记参数（314）将不同的行为彻底分开。

在函数间传递时，数据结构常会毫无必要地被拆开，我更愿意用保持对象完整（319）将其聚拢。函数需要的一份信息，究竟何时应该作为参数传入、何时应该调用一个函数获得，这是一个需要反复推敲的决定，推敲的过程中常常要用到以查询取代参数（324）和以参数取代查询（327）。

类是一种常见的模块形式。我希望尽可能保持对象不可变，所以只要有可能，我就会使用移除设值函数（331）。当调用者要求一个新对象时，我经常需要比构造函数更多的灵活性，可以借助以工厂函数取代构造函数（334）获得这种灵活性。

有时你会遇到一个特别复杂的函数，围绕着它传入传出一大堆数据。最后两个重构手法专门用于破解这个难题。我可以用以命令取代函数（337）将这个函数变成对象，这样对函数体使用提炼函数（106）时会更容易。如果稍后我对该函数做了简化，不再需要将其作为命令对象了，可以用以函数取代命令（344）再把它变回函数。

## 11.1 将查询函数和修改函数分离（Separate Query from Modifier）

```js
function getTotalOutstandingAndSendBill() {
const result = customer.invoices.reduce((total, each) => each.amount + total, 0);
sendBill();
return result;
}


function totalOutstanding() {
return customer.invoices.reduce((total, each) => each.amount + total, 0);
}
function sendBill() {
emailGateway.send(formatBill(customer));
}
```

### 动机

如果某个函数只是提供一个值，没有任何看得到的副作用，那么这是一个很有价值的东西。我可以任意调用这个函数，也可以把调用动作搬到调用函数的其他地方。这种函数的测试也更容易。简而言之，需要操心的事情少多了。

明确表现出“有副作用”与“无副作用”两种函数之间的差异，是个很好的想法。下面是一条好规则：任何有返回值的函数，都不应该有看得到的副作用——命令与查询分离（Command-Query Separation）[mf-cqs]。有些程序员甚至将此作为一条必须遵守的规则。就像对待任何东西一样，我并不绝对遵守它，不过我总是尽量遵守，而它也回报我很好的效果。

如果遇到一个“既有返回值又有副作用”的函数，我就会试着将查询动作从修改动作中分离出来。

你也许已经注意到了：我使用“看得到的副作用”这种说法。有一种常见的优化办法是：将查询所得结果缓存于某个字段中，这样一来后续的重复查询就可以大大加快速度。虽然这种做法改变了对象中缓存的状态，但这一修改是察觉不到的，因为不论如何查询，总是获得相同结果。

### 做法

复制整个函数，将其作为一个查询来命名。

如果想不出好名字，可以看看函数返回的是什么。查询的结果会被填入一个变量，这个变量的名字应该能对函数如何命名有所启发。

从新建的查询函数中去掉所有造成副作用的语句。

执行静态检查。

查找所有调用原函数的地方。如果调用处用到了该函数的返回值，就将其改为调用新建的查询函数，并在下面马上再调用一次原函数。每次修改之后都要测试。

从原函数中去掉返回值。

测试。

完成重构之后，查询函数与原函数之间常会有重复代码，可以做必要的清理。

### 范例

有这样一个函数：它会遍历一份恶棍（miscreant）名单，检查一群人（people）里是否混进了恶棍。如果发现了恶棍，该函数会返回恶棍的名字，并拉响警报。如果人群中有多名恶棍，该函数也只汇报找出的第一名恶棍（我猜这就已经够了）。

```js
function alertForMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

首先我复制整个函数，用它的查询部分功能为其命名。

```js
function findMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

然后在新建的查询函数中去掉副作用。

```js
function findMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

然后找到所有原函数的调用者，将其改为调用新建的查询函数，并在其后调用一次修改函数（也就是原函数）。于是代码

```js
const found = alertForMiscreant(people);
```

就变成了

```js
const found = findMiscreant(people);
alertForMiscreant(people);
```

现在可以从修改函数中去掉所有返回值了。

```js
function alertForMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return;
    }
    if (p === "John") {
      setOffAlarms();
      return;
    }
  }
  return;
}
```

现在，原来的修改函数和新建的查询函数之间有大量的重复代码，我可以使用替换算法（195），让修改函数使用查询函数。

```js
function alertForMiscreant(people) {
  if (findMiscreant(people) !== "") setOffAlarms();
}
```

## 11.2 函数参数化（Parameterize Function）

曾用名：令函数携带参数（Parameterize Method）

```js
function tenPercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.1);
}
function fivePercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.05);
}

function raise(aPerson, factor) {
  aPerson.salary = aPerson.salary.multiply(1 + factor);
}
```

### 动机

如果我发现两个函数逻辑非常相似，只有一些字面量值不同，可以将其合并成一个函数，以参数的形式传入不同的值，从而消除重复。这个重构可以使函数更有用，因为重构后的函数还可以用于处理其他的值。

### 做法

从一组相似的函数中选择一个。

运用改变函数声明（124），把需要作为参数传入的字面量添加到参数列表中。

修改该函数所有的调用处，使其在调用时传入该字面量值。

测试。

修改函数体，令其使用新传入的参数。每使用一个新参数都要测试。

对于其他与之相似的函数，逐一将其调用处改为调用已经参数化的函数。每次修改后都要测试。

如果第一个函数经过参数化以后不能直接替代另一个与之相似的函数，就先对参数化之后的函数做必要的调整，再做替换。

### 范例

下面是一个显而易见的例子：

```js
function tenPercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.1);
}
function fivePercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.05);
}
```

很明显我可以用下面这个函数来替换上面两个：

```js
function raise(aPerson, factor) {
  aPerson.salary = aPerson.salary.multiply(1 + factor);
}
```

情况可能比这个更复杂一些。例如下列代码：

```js
function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    bottomBand(usage) * 0.03
    + middleBand(usage) * 0.05
    + topBand(usage) * 0.07;
 return usd(amount);
}

function bottomBand(usage) {
 return Math.min(usage, 100);
}

function middleBand(usage) {
 return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function topBand(usage) {
 return usage > 200 ? usage - 200 : 0;
}
```

这几个函数中的逻辑明显很相似，但是不是相似到足以支撑一个参数化的计算“计费档次”（band）的函数？这次就不像前面第一个例子那样一目了然了。

在尝试对几个相关的函数做参数化操作时，我会先从中挑选一个，在上面添加参数，同时留意其他几种情况。在类似这样处理“范围”的情况下，通常从位于中间的范围开始着手较好。所以我首先选择了 middleBand 函数来添加参数，然后调整其他的调用者来适应它。

middleBand 使用了两个字面量值，即 100 和 200，分别代表“中间档次”的下界和上界。我首先用改变函数声明（124）加上这两个参数，同时顺手给函数改个名，使其更好地表述参数化之后的含义。

```js
function withinBand(usage, bottom, top) {
 return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    bottomBand(usage) * 0.03
    + withinBand(usage, 100, 200) * 0.05
    + topBand(usage) * 0.07;
 return usd(amount);
}
```

在函数体内部，把一个字面量改为使用新传入的参数：

```js
function withinBand(usage, bottom, top) {
  return usage & gt;
  bottom ? Math.min(usage, 200) - bottom : 0;
}
```

然后是另一个：

```js
function withinBand(usage, bottom, top) {
  return usage & gt;
  bottom ? Math.min(usage, top) - bottom : 0;
}
```

对于原本调用 bottomBand 函数的地方，我将其改为调用参数化了的新函数。

```js
function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    withinBand(usage, 0, 100) * 0.03
    + withinBand(usage, 100, 200) * 0.05
    + topBand(usage) * 0.07;
 return usd(amount);
}

function bottomBand(usage) {
 return Math.min(usage, 100);
}
```

为了替换对 topBand 的调用，我就得用代表“无穷大”的 Infinity 作为这个范围的上界。

```js
function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    withinBand(usage, 0, 100) * 0.03
    + withinBand(usage, 100, 200) * 0.05
    + withinBand(usage, 200, Infinity) * 0.07;
 return usd(amount);
}

function topBand(usage) {
  return usage > 200 ? usage - 200 : 0;
}
```

照现在的逻辑，baseCharge 一开始的卫语句已经可以去掉了。不过，尽管这条语句已经失去了逻辑上的必要性，我还是愿意把它留在原地，因为它阐明了“传入的 usage 参数为负数”这种情况是如何处理的。

## 11.3 移除标记参数（Remove Flag Argument）

曾用名：以明确函数取代参数（Replace Parameter with Explicit Methods）

```js
function setDimension(name, value) {
  if (name === "height") {
    this._height = value;
    return;
  }
  if (name === "width") {
    this._width = value;
    return;
  }
}

function setHeight(value) {
  this._height = value;
}
function setWidth(value) {
  this._width = value;
}
```

### 动机

“标记参数”是这样的一种参数：调用者用它来指示被调函数应该执行哪一部分逻辑。例如，我可能有下面这样一个函数：

```js
function bookConcert(aCustomer, isPremium) {
  if (isPremium) {
    // logic for premium booking
  } else {
    // logic for regular booking
  }
}
```

要预订一场高级音乐会（premium concert），就得这样发起调用：

```js
bookConcert(aCustomer, true);
```

标记参数也可能以枚举的形式出现：

```js
bookConcert(aCustomer, CustomerType.PREMIUM);
```

或者是以字符串（或者符号，如果编程语言支持的话）的形式出现：

```js
bookConcert(aCustomer, "premium");
```

我不喜欢标记参数，因为它们让人难以理解到底有哪些函数可以调用、应该怎么调用。拿到一份 API 以后，我首先看到的是一系列可供调用的函数，但标记参数却隐藏了函数调用中存在的差异性。使用这样的函数，我还得弄清标记参数有哪些可用的值。布尔型的标记尤其糟糕，因为它们不能清晰地传达其含义——在调用一个函数时，我很难弄清 true 到底是什么意思。如果明确用一个函数来完成一项单独的任务，其含义会清晰得多。

```js
premiumBookConcert(aCustomer);
```

并非所有类似这样的参数都是标记参数。如果调用者传入的是程序中流动的数据，这样的参数不算标记参数；只有调用者直接传入字面量值，这才是标记参数。另外，在函数实现内部，如果参数值只是作为数据传给其他函数，这就不是标记参数；只有参数值影响了函数内部的控制流，这才是标记参数。

移除标记参数不仅使代码更整洁，并且能帮助开发工具更好地发挥作用。去掉标记参数后，代码分析工具能更容易地体现出“高级”和“普通”两种预订逻辑在使用时的区别。

如果一个函数有多个标记参数，可能就不得不将其保留，否则我就得针对各个参数的各种取值的所有组合情况提供明确函数。不过这也是一个信号，说明这个函数可能做得太多，应该考虑是否能用更简单的函数来组合出完整的逻辑。

### 做法

针对参数的每一种可能值，新建一个明确函数。

如果主函数有清晰的条件分发逻辑，可以用分解条件表达式（260）创建明确函数；否则，可以在原函数之上创建包装函数。

对于“用字面量值作为参数”的函数调用者，将其改为调用新建的明确函数。

### 范例

在浏览代码时，我发现多处代码在调用一个函数计算物流（shipment）的到货日期（delivery date）。一些调用代码类似这样：

```js
aShipment.deliveryDate = deliveryDate(anOrder, true);
```

另一些调用代码则是这样：

```js
aShipment.deliveryDate = deliveryDate(anOrder, false);
```

面对这样的代码，我立即开始好奇：参数里这个布尔值是什么意思？是用来干什么的？

deliveryDate 函数主体如下所示：

```js
function deliveryDate(anOrder, isRush) {
  if (isRush) {
    let deliveryTime;
    if (["MA", "CT"].includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
  } else {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"].includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
  }
}
```

原来调用者用这个布尔型字面量来判断应该运行哪个分支的代码——典型的标记参数。然而函数的重点就在于要遵循调用者的指令，所以最好是用明确函数的形式明确说出调用者的意图。

对于这个例子，我可以使用分解条件表达式（260），得到下列代码：

```js
function deliveryDate(anOrder, isRush) {
  if (isRush) return rushDeliveryDate(anOrder);
  else return regularDeliveryDate(anOrder);
}
function rushDeliveryDate(anOrder) {
  let deliveryTime;
  if (["MA", "CT"].includes(anOrder.deliveryState)) deliveryTime = 1;
  else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
  else deliveryTime = 3;
  return anOrder.placedOn.plusDays(1 + deliveryTime);
}
function regularDeliveryDate(anOrder) {
  let deliveryTime;
  if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
  else if (["ME", "NH"].includes(anOrder.deliveryState)) deliveryTime = 3;
  else deliveryTime = 4;
  return anOrder.placedOn.plusDays(2 + deliveryTime);
}
```

这两个函数能更好地表达调用者的意图，现在我可以修改调用方代码了。调用代码

```js
aShipment.deliveryDate = deliveryDate(anOrder, true);
```

可以改为

```js
aShipment.deliveryDate = rushDeliveryDate(anOrder);
```

另一个分支也类似。

处理完所有调用处，我就可以移除 deliveryDate 函数。

这个参数是标记参数，不仅因为它是布尔类型，而且还因为调用方以字面量的形式直接设置参数值。如果所有调用 deliveryDate 的代码都像这样：

```js
const isRush = determineIfRush(anOrder);
aShipment.deliveryDate = deliveryDate(anOrder, isRush);
```

那我对这个函数的签名没有任何意见（不过我还是想用分解条件表达式（260）清理其内部实现）。

可能有一些调用者给这个参数传入的是字面量，将其作为标记参数使用；另一些调用者则传入正常的数据。若果真如此，我还是会使用移除标记参数（314），但不修改传入正常数据的调用者，重构结束时也不删除 deliveryDate 函数。这样我就提供了两套接口，分别支持不同的用途。

直接拆分条件逻辑是实施本重构的好方法，但只有当“根据参数值做分发”的逻辑发生在函数最外层（或者可以比较容易地将其重构至函数最外层）的时候，这一招才好用。函数内部也有可能以一种更纠结的方式使用标记参数，例如下面这个版本的 deliveryDate 函数：

```js
function deliveryDate(anOrder, isRush) {
 let result;
 let deliveryTime;
 if (anOrder.deliveryState === "MA" || anOrder.deliveryState === "CT")
  deliveryTime = isRush? 1 : 2;
 else if (anOrder.deliveryState === "NY" || anOrder.deliveryState === "NH") {
  deliveryTime = 2;
  if (anOrder.deliveryState === "NH" &amp;&amp; !isRush)
   deliveryTime = 3;
 }
 else if (isRush)
  deliveryTime = 3;
 else if (anOrder.deliveryState === "ME")
  deliveryTime = 3;
 else
  deliveryTime = 4;
 result = anOrder.placedOn.plusDays(2 + deliveryTime);
 if (isRush) result = result.minusDays(1);
 return result;
}
```

这种情况下，想把围绕 isRush 的分发逻辑剥离到顶层，需要的工作量可能会很大。所以我选择退而求其次，在 deliveryDate 之上添加两个函数：

```js
function rushDeliveryDate(anOrder) {
  return deliveryDate(anOrder, true);
}
function regularDeliveryDate(anOrder) {
  return deliveryDate(anOrder, false);
}
```

本质上，这两个包装函数分别代表了 deliveryDate 函数一部分的使用方式。不过它们并非从原函数中拆分而来，而是用代码文本强行定义的。

随后，我同样可以逐一替换原函数的调用者，就跟前面分解条件表达式之后的处理一样。如果没有任何一个调用者向 isRush 参数传入正常的数据，我最后会限制原函数的可见性，或是将其改名（例如改为 deliveryDateHelperOnly），让人一见即知不应直接使用这个函数。

## 11.4 保持对象完整（Preserve Whole Object）

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (aPlan.withinRange(low, high))


if (aPlan.withinRange(aRoom.daysTempRange))
```

### 动机

如果我看见代码从一个记录结构中导出几个值，然后又把这几个值一起传递给一个函数，我会更愿意把整个记录传给这个函数，在函数体内部导出所需的值。

“传递整个记录”的方式能更好地应对变化：如果将来被调的函数需要从记录中导出更多的数据，我就不用为此修改参数列表。并且传递整个记录也能缩短参数列表，让函数调用更容易看懂。如果有很多函数都在使用记录中的同一组数据，处理这部分数据的逻辑常会重复，此时可以把这些处理逻辑搬移到完整对象中去。

也有时我不想采用本重构手法，因为我不想让被调函数依赖完整对象，尤其是在两者不在同一个模块中的时候。

从一个对象中抽取出几个值，单独对这几个值做某些逻辑操作，这是一种代码坏味道（依恋情结），通常标志着这段逻辑应该被搬移到对象中。保持对象完整经常发生在引入参数对象（140）之后，我会搜寻使用原来的数据泥团的代码，代之以使用新的对象。

如果几处代码都在使用对象的一部分功能，可能意味着应该用提炼类（182）把这一部分功能单独提炼出来。

还有一种常被忽视的情况：调用者将自己的若干数据作为参数，传递给被调用函数。这种情况下，我可以将调用者的自我引用（在 JavaScript 中就是 this）作为参数，直接传递给目标函数。

### 做法

新建一个空函数，给它以期望中的参数列表（即传入完整对象作为参数）。

给这个函数起一个容易搜索的名字，这样到重构结束时方便替换。

在新函数体内调用旧函数，并把新的参数（即完整对象）映射到旧的参数列表（即来源于完整对象的各项数据）。

执行静态检查。

逐一修改旧函数的调用者，令其使用新函数，每次修改之后执行测试。

修改之后，调用处用于“从完整对象中导出参数值”的代码可能就没用了，可以用移除死代码（237）去掉。

所有调用处都修改过来之后，使用内联函数（115）把旧函数内联到新函数体内。

给新函数改名，从重构开始时的容易搜索的临时名字，改为使用旧函数的名字，同时修改所有调用处。

### 范例

我们想象一个室温监控系统，它负责记录房间一天中的最高温度和最低温度，然后将实际的温度范围与预先规定的温度控制计划（heating plan）相比较，如果当天温度不符合计划要求，就发出警告。

#### 调用方...

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.withinRange(low, high))
  alerts.push("room temperature went outside range");
```

#### class HeatingPlan...

```js
withinRange(bottom, top) {
 return (bottom >= this._temperatureRange.low) &amp;&amp; (top <= this._temperatureRange.high);
}
```

其实我不必将“温度范围”的信息拆开来单独传递，只需将整个范围对象传递给 withinRange 函数即可。

首先，我在 HeatingPlan 类中新添一个空函数，给它赋予我认为合理的参数列表。

#### class HeatingPlan...

```js
xxNEWwithinRange(aNumberRange) {
}
```

因为这个函数最终要取代现有的 withinRange 函数，所以它也用了同样的名字，再加上一个容易替换的前缀。

然后在新函数体内调用现有的 withinRange 函数。因此，新函数体就完成了从新参数列表到旧函数参数列表的映射。

#### class HeatingPlan...

```js
xxNEWwithinRange(aNumberRange) {
  return this.withinRange(aNumberRange.low, aNumberRange.high);
}
```

现在开始正式的替换工作了，我要找到调用现有函数的地方，将其改为调用新函数。

#### 调用方...

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.xxNEWwithinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

在修改调用处时，我可能会发现一些代码在修改后已经不再需要，此时可以使用移除死代码（237）。

#### 调用方...

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.xxNEWwithinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

每次替换一处调用代码，每次修改后都要测试。

调用处全部替换完成后，用内联函数（115）将旧函数内联到新函数体内。

#### class HeatingPlan...

```js
xxNEWwithinRange(aNumberRange) {
 return (aNumberRange.low >= this._temperatureRange.low) &amp;&amp;
  (aNumberRange.high <= this._temperatureRange.high);
}
```

终于可以去掉新函数那难看的前缀了，记得同时修改所有调用者。就算我所使用的开发环境不支持可靠的函数改名操作，有这个极具特色的前缀在，我也可以很方便地全局替换。

#### class HeatingPlan...

```js
withinRange(aNumberRange) {
 return (aNumberRange.low >= this._temperatureRange.low) &amp;&amp;
  (aNumberRange.high <= this._temperatureRange.high);
}
```

#### 调用方...

```js
if (!aPlan.withinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

### 范例：换个方式创建新函数

在上面的示例中，我直接编写了新函数。大多数时候，这一步非常简单，也是创建新函数最容易的方式。不过有时还会用到另一种方式：可以完全通过重构手法的组合来得到新函数。

我从一处调用现有函数的代码开始。

#### 调用方...

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.withinRange(low, high))
  alerts.push("room temperature went outside range");
```

我要先对代码做一些整理，以便用提炼函数（106）来创建新函数。目前的调用者代码还不具备可提炼的函数雏形，不过我可以先做几次提炼变量（119），使其轮廓显现出来。首先，我要把对旧函数的调用从条件判断中解放出来。

#### 调用方...

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
const isWithinRange = aPlan.withinRange(low, high);
if (!isWithinRange) alerts.push("room temperature went outside range");
```

然后把输入参数也提炼出来。

#### 调用方...

```js
const tempRange = aRoom.daysTempRange;
const low = tempRange.low;
const high = tempRange.high;
const isWithinRange = aPlan.withinRange(low, high);
if (!isWithinRange) alerts.push("room temperature went outside range");
```

完成这一步之后，就可以用提炼函数（106）来创建新函数。

#### 调用方...

```js
const tempRange = aRoom.daysTempRange;
const isWithinRange = xxNEWwithinRange(aPlan, tempRange);
if (!isWithinRange) alerts.push("room temperature went outside range");
```

#### 顶层作用域...

```js
function xxNEWwithinRange(aPlan, tempRange) {
  const low = tempRange.low;
  const high = tempRange.high;
  const isWithinRange = aPlan.withinRange(low, high);
  return isWithinRange;
}
```

由于旧函数属于另一个上下文（HeatingPlan 类），我需要用搬移函数（198）把新函数也搬过去。

#### 调用方...

```js
const tempRange = aRoom.daysTempRange;
const isWithinRange = aPlan.xxNEWwithinRange(tempRange);
if (!isWithinRange) alerts.push("room temperature went outside range");
```

#### class HeatingPlan...

```js
xxNEWwithinRange(tempRange) {
  const low = tempRange.low;
  const high = tempRange.high;
  const isWithinRange = this.withinRange(low, high);
  return isWithinRange;
}
```

剩下的过程就跟前面一样了：替换其他调用者，然后把旧函数内联到新函数中。重构刚开始的时候，为了清晰分离函数调用，以便提炼出新函数，我提炼了几个变量出来，现在可以把这些变量也内联回去。

这种方式的好处在于：它完全是由其他重构手法组合而成的。如果我使用的开发工具支持可靠的提炼和内联操作，用这种方式进行本重构会特别流畅。

## 11.5 以查询取代参数（Replace Parameter with Query）

曾用名：以函数取代参数（Replace Parameter with Method）

反向重构：以参数取代查询（327）

```js
availableVacation(anEmployee, anEmployee.grade);

function availableVacation(anEmployee, grade) {
  // calculate vacation...


  availableVacation(anEmployee)

function availableVacation(anEmployee) {
  const grade = anEmployee.grade;
  // calculate vacation...
```

### 动机

函数的参数列表应该总结该函数的可变性，标示出函数可能体现出行为差异的主要方式。和任何代码中的语句一样，参数列表应该尽量避免重复，并且参数列表越短就越容易理解。

如果调用函数时传入了一个值，而这个值由函数自己来获得也是同样容易，这就是重复。这个本不必要的参数会增加调用者的难度，因为它不得不找出正确的参数值，其实原本调用者是不需要费这个力气的。

“同样容易”四个字，划出了一条判断的界限。去除参数也就意味着“获得正确的参数值”的责任被转移：有参数传入时，调用者需要负责获得正确的参数值；参数去除后，责任就被转移给了函数本身。一般而言，我习惯于简化调用方，因此我愿意把责任移交给函数本身，但如果函数难以承担这份责任，就另当别论了。

不使用以查询取代参数最常见的原因是，移除参数可能会给函数体增加不必要的依赖关系——迫使函数访问某个程序元素，而我原本不想让函数了解这个元素的存在。这种“不必要的依赖关系”除了新增的以外，也可能是我想要稍后去除的，例如为了去除一个参数，我可能会在函数体内调用一个有问题的函数，或是从一个对象中获取某些原本想要剥离出去的数据。在这些情况下，都应该慎重考虑使用以查询取代参数。

如果想要去除的参数值只需要向另一个参数查询就能得到，这是使用以查询取代参数最安全的场景。如果可以从一个参数推导出另一个参数，那么几乎没有任何理由要同时传递这两个参数。

另外有一件事需要留意：如果在处理的函数具有引用透明性（referential transparency，即，不论任何时候，只要传入相同的参数值，该函数的行为永远一致），这样的函数既容易理解又容易测试，我不想使其失去这种优秀品质。我不会去掉它的参数，让它去访问一个可变的全局变量。

### 做法

如果有必要，使用提炼函数（106）将参数的计算过程提炼到一个独立的函数中。

将函数体内引用该参数的地方改为调用新建的函数。每次修改后执行测试。

全部替换完成后，使用改变函数声明（124）将该参数去掉。

### 范例

某些重构会使参数不再被需要，这是我最常用到以查询取代参数的场合。考虑下列代码。

#### class Order...

```js
get finalPrice() {
 const basePrice = this.quantity * this.itemPrice;
 let discountLevel;
 if (this.quantity > 100) discountLevel = 2;
 else discountLevel = 1;
 return this.discountedPrice(basePrice, discountLevel);
}

discountedPrice(basePrice, discountLevel) {
 switch (discountLevel) {
  case 1: return basePrice * 0.95;
  case 2: return basePrice * 0.9;
 }
}
```

在简化函数逻辑时，我总是热衷于使用以查询取代临时变量（178），于是就得到了如下代码。

#### class Order...

```js
get finalPrice() {
 const basePrice = this.quantity * this.itemPrice;
 return this.discountedPrice(basePrice, this.discountLevel);
}

get discountLevel() {
 return (this.quantity > 100) ? 2 : 1;
}
```

到这一步，已经不需要再把 discountLevel 的计算结果传给 discountedPrice 了，后者可以自己调用 discountLevel 函数，不会增加任何难度。

因此，我把 discountedPrice 函数中用到这个参数的地方全都改为直接调用 discountLevel 函数。

#### class Order...

```js
discountedPrice(basePrice, discountLevel) {
 switch (this.discountLevel) {
  case 1: return basePrice * 0.95;
  case 2: return basePrice * 0.9;
 }
}
```

然后用改变函数声明（124）手法移除该参数。

#### class Order...

```js
get finalPrice() {
 const basePrice = this.quantity * this.itemPrice;
 return this.discountedPrice(basePrice, this.discountLevel);
}

discountedPrice(basePrice, discountLevel) {
 switch (this.discountLevel) {
  case 1: return basePrice * 0.95;
  case 2: return basePrice * 0.9;
 }
}
```

## 11.6 以参数取代查询（Replace Query with Parameter）

反向重构：以查询取代参数（324）

```js
  targetTemperature(aPlan)

function targetTemperature(aPlan) {
  currentTemperature = thermostat.currentTemperature;
  // rest of function...


  targetTemperature(aPlan, thermostat.currentTemperature)

function targetTemperature(aPlan, currentTemperature) {
  // rest of function...
```

### 动机

在浏览函数实现时，我有时会发现一些令人不快的引用关系，例如，引用一个全局变量，或者引用另一个我想要移除的元素。为了解决这些令人不快的引用，我需要将其替换为函数参数，从而将处理引用关系的责任转交给函数的调用者。

需要使用本重构的情况大多源于我想要改变代码的依赖关系——为了让目标函数不再依赖于某个元素，我把这个元素的值以参数形式传递给该函数。这里需要注意权衡：如果把所有依赖关系都变成参数，会导致参数列表冗长重复；如果作用域之间的共享太多，又会导致函数间依赖过度。我一向不善于微妙的权衡，所以“能够可靠地改变决定”就显得尤为重要，这样随着我的理解加深，程序也能从中受益。

如果一个函数用同样的参数调用总是给出同样的结果，我们就说这个函数具有“引用透明性”（referential transparency），这样的函数理解起来更容易。如果一个函数使用了另一个元素，而后者不具引用透明性，那么包含该元素的函数也就失去了引用透明性。只要把“不具引用透明性的元素”变成参数传入，函数就能重获引用透明性。虽然这样就把责任转移给了函数的调用者，但是具有引用透明性的模块能带来很多益处。有一个常见的模式：在负责逻辑处理的模块中只有纯函数，其外再包裹处理 I/O 和其他可变元素的逻辑代码。借助以参数取代查询，我可以提纯程序的某些组成部分，使其更容易测试、更容易理解。

不过以参数取代查询并非只有好处。把查询变成参数以后，就迫使调用者必须弄清如何提供正确的参数值，这会增加函数调用者的复杂度，而我在设计接口时通常更愿意让接口的消费者更容易使用。归根到底，这是关于程序中责任分配的问题，而这方面的决策既不容易，也不会一劳永逸——这就是我需要非常熟悉本重构（及其反向重构）的原因。

### 做法

对执行查询操作的代码使用提炼变量（119），将其从函数体中分离出来。

现在函数体代码已经不再执行查询操作（而是使用前一步提炼出的变量），对这部分代码使用提炼函数（106）。

给提炼出的新函数起一个容易搜索的名字，以便稍后改名。

使用内联变量（123），消除刚才提炼出来的变量。

对原来的函数使用内联函数（115）。

对新函数改名，改回原来函数的名字。

### 范例

我们想象一个简单却又烦人的温度控制系统。用户可以从一个温控终端（thermostat）指定温度，但指定的目标温度必须在温度控制计划（heating plan）允许的范围内。

#### class HeatingPlan...

```js
get targetTemperature() {
  if (thermostat.selectedTemperature > this._max) return this._max;
  else if (thermostat.selectedTemperature < this._min) return this._min;
  else return thermostat.selectedTemperature;
}
```

#### 调用方...

```js
if (thePlan.targetTemperature > thermostat.currentTemperature) setToHeat();
else if (thePlan.targetTemperature<thermostat.currentTemperature)setToCool();
else setOff();
```

系统的温控计划规则抑制了我的要求，作为这样一个系统的用户，我可能会感到很烦恼。不过作为程序员，我更担心的是 targetTemperature 函数依赖于全局的 thermostat 对象。我可以把需要这个对象提供的信息作为参数传入，从而打破对该对象的依赖。

首先，我要用提炼变量（119）把“希望作为参数传入的信息”提炼出来。

#### class HeatingPlan...

```js
get targetTemperature() {
 const selectedTemperature = thermostat.selectedTemperature;
 if      (selectedTemperature > this._max) return this._max;
 else if (selectedTemperature < this._min) return this._min;
 else return selectedTemperature;
}
```

这样可以比较容易地用提炼函数（106）把整个函数体提炼出来，只剩“计算参数值”的逻辑还在原地。

#### class HeatingPlan...

```js
get targetTemperature() {
 const selectedTemperature = thermostat.selectedTemperature;
 return this.xxNEWtargetTemperature(selectedTemperature);
}

xxNEWtargetTemperature(selectedTemperature) {
 if      (selectedTemperature > this._max) return this._max;
 else if (selectedTemperature < this._min) return this._min;
 else return selectedTemperature;
}
```

然后把刚才提炼出来的变量内联回去，于是旧函数就只剩一个简单的调用。

#### class HeatingPlan...

```js
get targetTemperature() {
  return this.xxNEWtargetTemperature(thermostat.selectedTemperature);
}
```

现在可以对其使用内联函数（115）。

#### 调用方...

```js
if (thePlan.xxNEWtargetTemperature(thermostat.selectedTemperature) >
   thermostat.currentTemperature)
 setToHeat();
else if (thePlan.xxNEWtargetTemperature(thermostat.selectedTemperature) <
     thermostat.currentTemperature)
 setToCool();
else
 setOff();
```

再把新函数改名，用回旧函数的名字。得益于之前给它起了一个容易搜索的名字，现在只要把前缀去掉就行。

#### 调用方...

```js
if (thePlan.targetTemperature(thermostat.selectedTemperature) >
   thermostat.currentTemperature)
 setToHeat();
else if (thePlan.targetTemperature(thermostat.selectedTemperature) <
     thermostat.currentTemperature)
 setToCool();
else
 setOff();
```

#### class HeatingPlan...

```js
targetTemperature(selectedTemperature) {
 if (selectedTemperature > this._max) return this._max;
 else if (selectedTemperature < this._min) return this._min;
 else return selectedTemperature;
}
```

调用方的代码看起来比重构之前更笨重了，这是使用本重构手法的常见情况。将一个依赖关系从一个模块中移出，就意味着将处理这个依赖关系的责任推回给调用者。这是为了降低耦合度而付出的代价。

但是，去除对 thermostat 对象的耦合，并不是本重构带来的唯一收益。HeatingPlan 类本身是不可变的——字段的值都在构造函数中设置，任何函数都不会修改它们。（不用费心去查看整个类的代码，相信我就好。）在不可变的 HeatingPlan 基础上，把对 thermostat 的依赖移出函数体之后，我又使 targetTemperature 函数具备了引用透明性。从此以后，只要在同一个 HeatingPlan 对象上用同样的参数调用 targetTemperature 函数，我会始终得到同样的结果。如果 HeatingPlan 的所有函数都具有引用透明性，这个类会更容易测试，其行为也更容易理解。

JavaScript 的类模型有一个问题：无法强制要求类的不可变性——始终有办法修改对象的内部数据。尽管如此，在编写一个类的时候明确说明并鼓励不可变性，通常也就足够了。尽量让类保持不可变通常是一个好的策略，以参数取代查询则是达成这一策略的利器。

## 11.7 移除设值函数（Remove Setting Method）

```js
class Person {
get name() {...}
set name(aString) {...}


class Person {
get name() {...}
```

### 动机

如果为某个字段提供了设值函数，这就暗示这个字段可以被改变。如果不希望在对象创建之后此字段还有机会被改变，那就不要为它提供设值函数（同时将该字段声明为不可变）。这样一来，该字段就只能在构造函数中赋值，我“不想让它被修改”的意图会更加清晰，并且可以排除其值被修改的可能性——这种可能性往往是非常大的。

有两种常见的情况需要讨论。一种情况是，有些人喜欢始终通过访问函数来读写字段值，包括在构造函数内也是如此。这会导致构造函数成为设值函数的唯一使用者。若果真如此，我更愿意去除设值函数，清晰地表达“构造之后不应该再更新字段值”的意图。

另一种情况是，对象是由客户端通过创建脚本构造出来，而不是只有一次简单的构造函数调用。所谓“创建脚本”，首先是调用构造函数，然后就是一系列设值函数的调用，共同完成新对象的构造。创建脚本执行完以后，这个新生对象的部分（乃至全部）字段就不应该再被修改。设值函数只应该在起初的对象创建过程中调用。对于这种情况，我也会想办法去除设值函数，更清晰地表达我的意图。

### 做法

如果构造函数尚无法得到想要设入字段的值，就使用改变函数声明（124）将这个值以参数的形式传入构造函数。在构造函数中调用设值函数，对字段设值。

如果想移除多个设值函数，可以一次性把它们的值都传入构造函数，这能简化后续步骤。

移除所有在构造函数之外对设值函数的调用，改为使用新的构造函数。每次修改之后都要测试。

如果不能把“调用设值函数”替换为“创建一个新对象”（例如你需要更新一个多处共享引用的对象），请放弃本重构。

使用内联函数（115）消去设值函数。如果可能的话，把字段声明为不可变。

测试。

### 范例

我有一个很简单的 Person 类。

#### class Person...

```js
get name() {return this._name;}
set name(arg) {this._name = arg;}
get id() {return this._id;}
set id(arg) {this._id = arg;}
```

目前我会这样创建新对象：

```js
const martin = new Person();
martin.name = "martin";
martin.id = "1234";
```

对象创建之后，name 字段可能会改变，但 id 字段不会。为了更清晰地表达这个设计意图，我希望移除对应 id 字段的设值函数。

但 id 字段还得设置初始值，所以我首先用改变函数声明（124）在构造函数中添加对应的参数。

#### class Person...

```js
constructor(id) {
  this.id = id;
}
```

然后调整创建脚本，改为从构造函数设值 id 字段值。

```js
const martin = new Person("1234");
martin.name = "martin";
martin.id = "1234";
```

所有创建 Person 对象的地方都要如此修改，每次修改之后要执行测试。

全部修改完成后，就可以用内联函数（115）消去设值函数。

#### class Person...

```js
constructor(id) {
  this._id = id;
}
get name() {return this._name;}
set name(arg) {this._name = arg;}
get id() {return this._id;}
set id(arg) {this._id = arg;}
```

## 11.8 以工厂函数取代构造函数（Replace Constructor with Factory Function）

曾用名：以工厂函数取代构造函数（Replace Constructor with Factory Method）

```js
leadEngineer = new Employee(document.leadEngineer, "E");

leadEngineer = createEngineer(document.leadEngineer);
```

### 动机

很多面向对象语言都有特别的构造函数，专门用于对象的初始化。需要新建一个对象时，客户端通常会调用构造函数。但与一般的函数相比，构造函数又常有一些丑陋的局限性。例如，Java 的构造函数只能返回当前所调用类的实例，也就是说，我无法根据环境或参数信息返回子类实例或代理对象；构造函数的名字是固定的，因此无法使用比默认名字更清晰的函数名；构造函数需要通过特殊的操作符来调用（在很多语言中是 new 关键字），所以在要求普通函数的场合就难以使用。

工厂函数就不受这些限制。工厂函数的实现内部可以调用构造函数，但也可以换成别的方式实现。

### 做法

新建一个工厂函数，让它调用现有的构造函数。

将调用构造函数的代码改为调用工厂函数。

每修改一处，就执行测试。

尽量缩小构造函数的可见范围。

### 范例

又是那个单调乏味的例子：员工薪资系统。我还是以 Employee 类表示“员工”。

#### class Employee...

```js
constructor (name, typeCode) {
  this._name = name;
  this._typeCode = typeCode;
}
get name() {return this._name;}
get type() {
  return Employee.legalTypeCodes[this._typeCode];
}
static get legalTypeCodes() {
  return {"E": "Engineer", "M": "Manager", "S": "Salesman"};
}
```

使用它的代码有这样的：

#### 调用方...

```js
candidate = new Employee(document.name, document.empType);
```

也有这样的：

#### 调用方...

```js
const leadEngineer = new Employee(document.leadEngineer, "E");
```

重构的第一步是创建工厂函数，其中把对象创建的责任直接委派给构造函数。

#### 顶层作用域...

```js
function createEmployee(name, typeCode) {
  return new Employee(name, typeCode);
}
```

然后找到构造函数的调用者，并逐一修改它们，令其使用工厂函数。

第一处的修改很简单。

#### 调用方...

```js
candidate = createEmployee(document.name, document.empType);
```

第二处则可以这样使用工厂函数。

#### 调用方...

```js
const leadEngineer = createEmployee(document.leadEngineer, "E");
```

但我不喜欢这里的类型码——以字符串字面量的形式传入类型码，一般来说都是坏味道。所以我更愿意再新建一个工厂函数，把“员工类别”的信息嵌在函数名里体现。

#### 调用方...

```js
const leadEngineer = createEngineer(document.leadEngineer);
```

#### 顶层作用域...

```js
function createEngineer(name) {
  return new Employee(name, "E");
}
```

## 11.9 以命令取代函数（Replace Function with Command）

曾用名：以函数对象取代函数（Replace Method with Method Object）

反向重构：以函数取代命令（344）

```js
function score(candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  // long body code
}

class Scorer {
  constructor(candidate, medicalExam, scoringGuide) {
    this._candidate = candidate;
    this._medicalExam = medicalExam;
    this._scoringGuide = scoringGuide;
  }

  execute() {
    this._result = 0;
    this._healthLevel = 0;
    // long body code
  }
}
```

### 动机

函数，不管是独立函数，还是以方法（method）形式附着在对象上的函数，是程序设计的基本构造块。不过，将函数封装成自己的对象，有时也是一种有用的办法。这样的对象我称之为“命令对象”（command object），或者简称“命令”（command）。这种对象大多只服务于单一函数，获得对该函数的请求，执行该函数，就是这种对象存在的意义。

与普通的函数相比，命令对象提供了更大的控制灵活性和更强的表达能力。除了函数调用本身，命令对象还可以支持附加的操作，例如撤销操作。我可以通过命令对象提供的方法来设值命令的参数值，从而支持更丰富的生命周期管理能力。我可以借助继承和钩子对函数行为加以定制。如果我所使用的编程语言支持对象但不支持函数作为一等公民，通过命令对象就可以给函数提供大部分相当于一等公民的能力。同样，即便编程语言本身并不支持嵌套函数，我也可以借助命令对象的方法和字段把复杂的函数拆解开，而且在测试和调试过程中可以直接调用这些方法。

所有这些都是使用命令对象的好理由，所以我要做好准备，一旦有需要，就能把函数重构成命令。不过我们不能忘记，命令对象的灵活性也是以复杂性作为代价的。所以，如果要在作为一等公民的函数和命令对象之间做个选择，95%的时候我都会选函数。只有当我特别需要命令对象提供的某种能力而普通的函数无法提供这种能力时，我才会考虑使用命令对象。

跟软件开发中的很多词汇一样，“命令”这个词承载了太多含义。在这里，“命令”是指一个对象，其中封装了一个函数调用请求。这是遵循《设计模式》[gof]一书中的命令模式（command pattern）。在这个意义上，使用“命令”一词时，我会先用完整的“命令对象”一词设定上下文，然后视情况使用简略的“命令”一词。在命令与查询分离原则（command-query separation principle）中也用到了“命令”一词，此时“命令”是一个对象所拥有的函数，调用该函数可以改变对象可观察的状态。我尽量避免使用这个意义上的“命令”一词，而更愿意称其为“修改函数”（modifier）或者“改变函数”（mutator）。

### 做法

为想要包装的函数创建一个空的类，根据该函数的名字为其命名。

使用搬移函数（198）把函数移到空的类里。

保持原来的函数作为转发函数，至少保留到重构结束之前才删除。

遵循编程语言的命名规范来给命令对象起名。如果没有合适的命名规范，就给命令对象中负责实际执行命令的函数起一个通用的名字，例如“execute”或者“call”。

可以考虑给每个参数创建一个字段，并在构造函数中添加对应的参数。

### 范例

JavaScript 语言有很多缺点，但把函数作为一等公民对待，是它最正确的设计决策之一。在不具备这种能力的编程语言中，我经常要费力为很常见的任务创建命令对象，JavaScript 则省去了这些麻烦。不过，即便在 JavaScript 中，有时也需要用到命令对象。

一个典型的应用场景就是拆解复杂的函数，以便我理解和修改。要想真正展示这个重构手法的价值，我需要一个长而复杂的函数，但这写起来太费事，你读起来也麻烦。所以我在这里展示的函数其实很短，并不真的需要本重构手法，还望读者权且包涵。下面的函数用于给一份保险申请评分。

```js
function score(candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  let certificationGrade = "regular";
  if (scoringGuide.stateWithLowCertification(candidate.originState)) {
    certificationGrade = "low";
    result -= 5;
  } // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

我首先创建一个空的类，用搬移函数（198）把上述函数搬到这个类里去。

```js
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer().execute(candidate, medicalExam, scoringGuide);
}

class Scorer {
  execute(candidate, medicalExam, scoringGuide) {
    let result = 0;
    let healthLevel = 0;
    let highMedicalRiskFlag = false;

    if (medicalExam.isSmoker) {
      healthLevel += 10;
      highMedicalRiskFlag = true;
    }
    let certificationGrade = "regular";
    if (scoringGuide.stateWithLowCertification(candidate.originState)) {
      certificationGrade = "low";
      result -= 5;
    } // lots more code like this
    result -= Math.max(healthLevel - 5, 0);
    return result;
  }
}
```

大多数时候，我更愿意在命令对象的构造函数中传入参数，而不让 execute 函数接收参数。在这样一个简单的拆解场景中，这一点带来的影响不大；但如果我要处理的命令需要更复杂的参数设置周期或者大量定制，上述做法就会带来很多便利：多个命令类可以分别从各自的构造函数中获得各自不同的参数，然后又可以排成队列挨个执行，因为它们的 execute 函数签名都一样。

我可以每次搬移一个参数到构造函数。

```js
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer(candidate).execute(candidate, medicalExam, scoringGuide);
}
```

#### class Scorer...

```js
constructor(candidate){
 this._candidate = candidate;
}

execute (candidate, medicalExam, scoringGuide) {
 let result = 0;
 let healthLevel = 0;
 let highMedicalRiskFlag = false;

 if (medicalExam.isSmoker) {
  healthLevel += 10;
  highMedicalRiskFlag = true;
 }
 let certificationGrade = "regular";
 if (scoringGuide.stateWithLowCertification(this._candidate.originState)) {
  certificationGrade = "low";
  result -= 5;
 }
 // lots more code like this
 result -= Math.max(healthLevel - 5, 0);
 return result;
}
```

继续处理其他参数：

```js
function score(candidate, medicalExam, scoringGuide) {
  return new Scorer(candidate, medicalExam, scoringGuide).execute();
}
```

#### class Scorer...

```js
constructor(candidate, medicalExam, scoringGuide){
 this._candidate = candidate;
 this._medicalExam = medicalExam;
 this._scoringGuide = scoringGuide;
}
execute () {
 let result = 0;
 let healthLevel = 0;
 let highMedicalRiskFlag = false;

 if (this._medicalExam.isSmoker) {
  healthLevel += 10;
  highMedicalRiskFlag = true;
 }
 let certificationGrade = "regular";
 if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
  certificationGrade = "low";
  result -= 5;
 }
 // lots more code like this
 result -= Math.max(healthLevel - 5, 0);
 return result;
}
```

以命令取代函数的重构到此就结束了，不过之所以要做这个重构，是为了拆解复杂的函数，所以我还是大致展示一下如何拆解。下一步是把所有局部变量都变成字段，我还是每次修改一处。

#### class Scorer...

```js
constructor(candidate, medicalExam, scoringGuide){
 this._candidate = candidate;
 this._medicalExam = medicalExam;
 this._scoringGuide = scoringGuide;
}

execute () {
 this._result = 0;
 let healthLevel = 0;
 let highMedicalRiskFlag = false;

 if (this._medicalExam.isSmoker) {
  healthLevel += 10;
  highMedicalRiskFlag = true;
 }
 let certificationGrade = "regular";
 if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
  certificationGrade = "low";
  this._result -= 5;
 }
 // lots more code like this
 this._result -= Math.max(healthLevel - 5, 0);
 return this._result;
}
```

重复上述过程，直到所有局部变量都变成字段。（“把局部变量变成字段”这个重构手法是如此简单，以至于我都没有在重构名录中给它一席之地。对此我略感愧疚。）

#### class Scorer...

```js
constructor(candidate, medicalExam, scoringGuide){
 this._candidate = candidate;
 this._medicalExam = medicalExam;
 this._scoringGuide = scoringGuide;
}

execute () {
 this._result = 0;
 this._healthLevel = 0;
 this._highMedicalRiskFlag = false;

 if (this._medicalExam.isSmoker) {
  this._healthLevel += 10;
  this._highMedicalRiskFlag = true;
 }
 this._certificationGrade = "regular";
 if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
  this._certificationGrade = "low";
  this._result -= 5;
 }
 // lots more code like this
 this._result -= Math.max(this._healthLevel - 5, 0);
 return this._result;
}
```

现在函数的所有状态都已经移到了命令对象中，我可以放心使用提炼函数（106）等重构手法，而不用纠结于局部变量的作用域之类问题。

#### class Scorer...

```js
execute () {
 this._result = 0;
 this._healthLevel = 0;
 this._highMedicalRiskFlag = false;

 this.scoreSmoking();
 this._certificationGrade = "regular";
 if (this._scoringGuide.stateWithLowCertification(this._candidate.originState)) {
  this._certificationGrade = "low";
  this._result -= 5;
 }
 // lots more code like this
 this._result -= Math.max(this._healthLevel - 5, 0);
 return this._result;
 }
scoreSmoking() {
 if (this._medicalExam.isSmoker) {
  this._healthLevel += 10;
  this._highMedicalRiskFlag = true;
 }
}
```

这样我就可以像处理嵌套函数一样处理命令对象。实际上，在 JavaScript 中运用此重构手法时，的确可以考虑用嵌套函数来代替命令对象。不过我还是会使用命令对象，不仅因为我对命令对象更熟悉，而且还因为我可以针对命令对象中任何一个函数进行测试和调试。

## 11.10 以函数取代命令（Replace Command with Function）

反向重构：以命令取代函数（337）

```js
class ChargeCalculator {
  constructor(customer, usage) {
    this._customer = customer;
    this._usage = usage;
  }
  execute() {
    return this._customer.rate * this._usage;
  }
}

function charge(customer, usage) {
  return customer.rate * usage;
}
```

### 动机

命令对象为处理复杂计算提供了强大的机制。借助命令对象，可以轻松地将原本复杂的函数拆解为多个方法，彼此之间通过字段共享状态；拆解后的方法可以分别调用；开始调用之前的数据状态也可以逐步构建。但这种强大是有代价的。大多数时候，我只是想调用一个函数，让它完成自己的工作就好。如果这个函数不是太复杂，那么命令对象可能显得费而不惠，我就应该考虑将其变回普通的函数。

### 做法

运用提炼函数（106），把“创建并执行命令对象”的代码单独提炼到一个函数中。

这一步会新建一个函数，最终这个函数会取代现在的命令对象。

对命令对象在执行阶段用到的函数，逐一使用内联函数（115）。

如果被调用的函数有返回值，请先对调用处使用提炼变量（119），然后再使用内联函数（115）。

使用改变函数声明（124），把构造函数的参数转移到执行函数。

对于所有的字段，在执行函数中找到引用它们的地方，并改为使用参数。每次修改后都要测试。

把“调用构造函数”和“调用执行函数”两步都内联到调用方（也就是最终要替换命令对象的那个函数）。

测试。

用移除死代码（237）把命令类消去。

### 范例

假设我有一个很小的命令对象。

```js
class ChargeCalculator {
  constructor(customer, usage, provider) {
    this._customer = customer;
    this._usage = usage;
    this._provider = provider;
  }
  get baseCharge() {
    return this._customer.baseRate * this._usage;
  }
  get charge() {
    return this.baseCharge + this._provider.connectionCharge;
  }
}
```

使用方的代码如下。

#### 调用方...

```js
monthCharge = new ChargeCalculator(customer, usage, provider).charge;
```

命令类足够小、足够简单，变成函数更合适。

首先，我用提炼函数（106）把命令对象的创建与调用过程包装到一个函数中。

#### 调用方...

```js
monthCharge = charge(customer, usage, provider);
```

#### 顶层作用域...

```js
function charge(customer, usage, provider) {
  return new ChargeCalculator(customer, usage, provider).charge;
}
```

接下来要考虑如何处理支持函数（也就是这里的 baseCharge 函数）。对于有返回值的函数，我一般会先用提炼变量（119）把返回值提炼出来。

#### class ChargeCalculator...

```js
get baseCharge() {
  return this._customer.baseRate * this._usage;
}
get charge() {
  const baseCharge = this.baseCharge;
  return baseCharge + this._provider.connectionCharge;
}
```

然后对支持函数使用内联函数（115）。

#### class ChargeCalculator...

```js
get charge() {
  const baseCharge = this._customer.baseRate * this._usage;
  return baseCharge + this._provider.connectionCharge;
}
```

现在所有逻辑处理都集中到一个函数了，下一步是把构造函数传入的数据移到主函数。首先用改变函数声明（124）把构造函数的参数逐一添加到 charge 函数上。

#### class ChargeCalculator...

```js
constructor (customer, usage, provider){
 this._customer = customer;
 this._usage = usage;
 this._provider = provider;
}

charge(customer, usage, provider) {
 const baseCharge = this._customer.baseRate * this._usage;
 return baseCharge + this._provider.connectionCharge;
}
```

#### 顶层作用域...

```js
function charge(customer, usage, provider) {
  return new ChargeCalculator(customer, usage, provider).charge(
    customer,
    usage,
    provider
  );
}
```

然后修改 charge 函数的实现，改为使用传入的参数。这个修改可以小步进行，每次使用一个参数。

#### class ChargeCalculator...

```js
constructor (customer, usage, provider){
 this._customer = customer;
 this._usage = usage;
 this._provider = provider;
}

charge(customer, usage, provider) {
 const baseCharge = customer.baseRate * this._usage;
 return baseCharge + this._provider.connectionCharge;
}
```

构造函数中对 `this._customer` 字段的赋值不删除也没关系，因为反正没人使用这个字段。但我更愿意去掉这条赋值语句，因为去掉它以后，如果在函数实现中漏掉了一处对字段的使用没有修改，测试就会失败。（如果我真的犯了这个错误而测试没有失败，我就应该考虑增加测试了。）

其他参数也如法炮制，直到 charge 函数不再使用任何字段：

#### class ChargeCalculator...

```js
charge(customer, usage, provider) {
  const baseCharge = customer.baseRate * usage;
  return baseCharge + provider.connectionCharge;
}
```

现在我就可以把所有逻辑都内联到顶层的 charge 函数中。这是内联函数（115）的一种特殊情况，我需要把构造函数和执行函数一并内联。

#### 顶层作用域...

```js
function charge(customer, usage, provider) {
  const baseCharge = customer.baseRate * usage;
  return baseCharge + provider.connectionCharge;
}
```

现在命令类已经是死代码了，可以用移除死代码（237）给它一个体面的葬礼。
