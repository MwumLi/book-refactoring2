# 第 8 章 搬移特性

到目前为止，我介绍的重构手法都是关于如何新建、移除或重命名程序的元素。此外，还有另一种类型的重构也很重要，那就是在不同的上下文之间搬移元素。我会通过搬移函数（198）手法在类与其他模块之间搬移函数，对于字段可用搬移字段（207）手法做类似的搬移。

有时我还需要单独对语句进行搬移，调整它们的顺序。搬移语句到函数（213）和搬移语句到调用者（217）可用于将语句搬入函数或从函数中搬出；如果需要在函数内部调整语句的顺序，那么移动语句（223）就能派上用场。有时一些语句做的事已有现成的函数代替，那时我就能以函数调用取代内联代码（222）消除重复。

对付循环，我有两个常用的手法：拆分循环（227）可以确保每个循环只做一件事，以管道取代循环（231）则可以直接消灭整个循环。

最后这项手法，我相信一定会是任何一个合格程序员的至爱，那就是移除死代码（237）。没什么能比手刃一段长长的无用代码更令一个程序员感到满足的了。

## 8.1 搬移函数（Move Function）

曾用名：搬移函数（Move Method）

```js
class Account {
get overdraftCharge() {...}


class AccountType {
  get overdraftCharge() {...}
```

### 动机

模块化是优秀软件设计的核心所在，好的模块化能够让我在修改程序时只需理解程序的一小部分。为了设计出高度模块化的程序，我得保证互相关联的软件要素都能集中到一块，并确保块与块之间的联系易于查找、直观易懂。同时，我对模块设计的理解并不是一成不变的，随着我对代码的理解加深，我会知道那些软件要素如何组织最为恰当。要将这种理解反映到代码上，就得不断地搬移这些元素。

任何函数都需要具备上下文环境才能存活。这个上下文可以是全局的，但它更多时候是由某种形式的模块所提供的。对一个面向对象的程序而言，类作为最主要的模块化手段，其本身就能充当函数的上下文；通过嵌套的方式，外层函数也能为内层函数提供一个上下文。不同的语言提供的模块化机制各不相同，但这些模块的共同点是，它们都能为函数提供一个赖以存活的上下文环境。

搬移函数最直接的一个动因是，它频繁引用其他上下文中的元素，而对自身上下文中的元素却关心甚少。此时，让它去与那些更亲密的元素相会，通常能取得更好的封装效果，因为系统别处就可以减少对当前模块的依赖。

同样，如果我在整理代码时，发现需要频繁调用一个别处的函数，我也会考虑搬移这个函数。有时你在函数内部定义了一个帮助函数，而该帮助函数可能在别的地方也有用处，此时就可以将它搬移到某些更通用的地方。同理，定义在一个类上的函数，可能挪到另一个类中去更方便我们调用。

是否需要搬移函数常常不易抉择。为了做出决定，我需要仔细检查函数当前上下文与目标上下文之间的区别，需要查看函数的调用者都有谁，它自身又调用了哪些函数，被调用函数需要什么数据，等等。在搬移过程中，我通常会发现需要为一整组函数创建一个新的上下文，此时就可以用函数组合成类（144）或提炼类（182）创建一个。尽管为函数选择一个最好的去处不太容易，但决定越难做，通常说明“搬移这个函数与否”的重要性也越低。我发现好的做法是先把函数安置到某一个上下文里去，这样我就能发现它们是否契合，如果不太合适我可以再把函数搬移到别的地方。

### 做法

检查函数在当前上下文里引用的所有程序元素（包括变量和函数），考虑是否需要将它们一并搬移

如果发现有些被调用的函数也需要搬移，我通常会先搬移它们。这样可以保证移动一组函数时，总是从依赖最少的那个函数入手。

如果该函数拥有一些子函数，并且它是这些子函数的唯一调用者，那么你可以先将子函数内联进来，一并搬移到新家后再重新提炼出子函数。

检查待搬移函数是否具备多态性。

在面向对象的语言里，还需要考虑该函数是否覆写了超类的函数，或者为子类所覆写。

将函数复制一份到目标上下文中。调整函数，使它能适应新家。

如果函数里用到了源上下文（source context）中的元素，我就得将这些元素一并传递过去，要么通过函数参数，要么是将当前上下文的引用传递到新的上下文那边去。

搬移函数通常意味着，我还得给它起个新名字，使它更符合新的上下文。

执行静态检查。

设法从源上下文中正确引用目标函数。

修改源函数，使之成为一个纯委托函数。

测试。

考虑对源函数使用内联函数（115）

也可以不做内联，让源函数一直做委托调用。但如果调用方直接调用目标函数也不费太多周折，那么最好还是把中间人移除掉。

### 范例：搬移内嵌函数至顶层

让我用一个函数来举例。这个函数会计算一条 GPS 轨迹记录（track record）的总距离（total distance）。

```js
function trackSummary(points) {
 const totalTime = calculateTime();
 const totalDistance = calculateDistance();
 const pace = totalTime / 60 / totalDistance ;
 return {
  time: totalTime,
  distance: totalDistance,
  pace: pace
 };

 function calculateDistance() {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
   result += distance(points[i-1],  points[i]);
  }
  return result;
 }

 function distance(p1,p2) { ... }
 function radians(degrees) { ... }
 function calculateTime() { ... }

}
```

我希望把 calculateDistance 函数搬移到顶层，这样我就能单独计算轨迹的距离，而不必算出汇总报告（summary）的其他部分。

我先将函数复制一份到顶层作用域中：

```js
  function trackSummary(points) {
 const totalTime = calculateTime();
 const totalDistance = calculateDistance();
 const pace = totalTime / 60 / totalDistance ;
 return {
  time: totalTime,
  distance: totalDistance,
  pace: pace
 };

 function calculateDistance() {
  let result  =  0;
  for (let i = 1; i < points.length; i++) {
   result += distance(points[i-1], points[i]);
  }
  return result;
 }
 ...
 function distance(p1,p2) { ... }
 function radians(degrees) { ... }
 function calculateTime() { ... }

}

 function top_calculateDistance() {
  let result  =  0;
  for (let i = 1; i < points.length; i++) {
   result += distance(points[i-1],  points[i]);
  }
  return result;
 }
```

复制函数时，我习惯为函数一并改个名，这样可让“它们有不同的作用域”这个信息显得一目了然。现在我还不想花费心思考虑它正确的名字该是什么，因此我暂且先用一个临时的名字。

此时代码依然能正常工作，但我的静态分析器要开始抱怨了，它说新函数里多了两个未定义的符号，分别是 distance 和 points。对于 points，自然是将其作为函数参数传进来。

```js
  function top_calculateDistance(points) {
 let result =0;
 for (let i = 1; i < points.length; i++) {
  result += distance(points[i-1],  points[i]);
 }
 return result;
}
```

至于 distance，虽然我也可以将它作为参数传进来，但也许将其计算函数 calculate Distance 一并搬移过来会更合适。该函数的代码如下。

#### function trackSummary...

```js
function distance(p1, p2) {
  const EARTH_RADIUS = 3959; // in miles
  const dLat = radians(p2.lat) - radians(p1.lat);
  const dLon = radians(p2.lon) - radians(p1.lon);
  const a =
    Math.pow(Math.sin(dLat / 2), 2) +
    Math.cos(radians(p2.lat)) *
      Math.cos(radians(p1.lat)) *
      Math.pow(Math.sin(dLon / 2), 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return EARTH_RADIUS * c;
}
function radians(degrees) {
  return (degrees * Math.PI) / 180;
}
```

我留意到 distance 函数中只调用了 radians 函数，后者已经没有再引用当前上下文里的任何元素。因此与其将 radians 作为参数，我更倾向于将它也一并搬移。不过我不需要一步到位，我们可以先将这两个函数从当前上下文中搬移进 calculateDistance 函数里：

```js
  function trackSummary(points) {
 const totalTime = calculateTime();
 const totalDistance = calculateDistance();
 const pace = totalTime / 60 / totalDistance ;
 return {
  time: totalTime,
  distance: totalDistance,
  pace: pace
 };

 function calculateDistance() {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
   result += distance(points[i-1], points[i]);
  }
  return result;

  function distance(p1,p2) { ... }
  function radians(degrees) { ... }

}
```

这样做的好处是，我可以充分发挥静态检查和测试的作用，让它们帮我检查有无遗漏的东西。在这个实例中一切顺利，因此，我可以放心地将这两个函数直接复制到 top_calculateDistance 中：

```js
  function top_calculateDistance(points) {
 let result = 0;
 for (let i = 1; i < points.length; i++) {
  result += distance(points[i-1],  points[i]);
 }
 return result;

 function distance(p1,p2) { ... }
 function radians(degrees) { ... }

}
```

这次复制操作同样不会改变程序现有行为，但给了静态分析器更多介入的机会，增加了暴露错误的概率。假如我在上一步没有发现 distance 函数内部还调用了 radians 函数，那么这一步就会被分析器检查出来。

现在万事俱备，是时候端出主菜了——我要在原 calculateDistance 函数体内调用 top_calculateDistance 函数：

```js
  function trackSummary(points) {
 const totalTime = calculateTime();
 const totalDistance = calculateDistance();
 const pace = totalTime / 60 / totalDistance ;
 return {
  time: totalTime,
  distance: totalDistance,
  pace: pace
 };

 function calculateDistance() {
  return top_calculateDistance(points);
 }
```

接下来最重要的事是要运行一遍测试，看看功能是否仍然完整，函数在其新家待得是否舒适。

测试通过后，便算完成了主要任务，就好比搬家，现在大箱小箱已经全搬到新家，接下来就是将它们拆箱复位了。第一件事是决定还要不要保留原来那个只起委托作用的函数。在这个例子中，原函数的调用点不多，作为嵌套函数它们的作用范围通常也很小，因此我觉得这里大可直接移除原函数。

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = top_calculateDistance(points);
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace,
  };
}
```

同时，也该是时候为这个函数认真想个名字了。因为顶层函数拥有最高的可见性，因此取个好名非常重要。totalDistance 听起来不错，但还不能马上用这个名字，因为 trackSummary 函数中有一个同名的变量——我不觉得这个变量有保留的价值，因此我们先用内联变量（123）处理它，之后再使用改变函数声明（124）：

```js
  function trackSummary(points) {
 const totalTime = calculateTime();
 const pace = totalTime / 60 / totalDistance(points) ;
 return {
  time: totalTime,
  distance: totalDistance(points),
  pace: pace
 };
}
function totalDistance(points) {
 let result = 0;
 for (let i = 1; i < points.length; i++) {
  result += distance(points[i-1], points[i]);
 }
 return result;
}
```

如果出于某些原因，实在需要保留该变量，那么我建议将该变量改个其他的名字，比如 totalDistanceCache 或 distance 等。

由于 distance 函数和 radians 函数并未使用 totalDistance 中的任何变量或函数，因此我倾向于把它们也提升到顶层，也就是 4 个方法都放置在顶层作用域上。

```js
  function trackSummary(points) { ... }
function totalDistance(points) { ... }
function distance(p1,p2) { ... }
function radians(degrees) { ... }
```

有些人则更倾向于将 distance 和 radians 函数保留在 totalDistance 内，以便限制它们的可见性。在某些语言里，这个顾虑也许有其道理，但新的 ES 2015 规范为 JavaScript 提供了一个美妙的模块化机制，利用它来控制函数的可见性是再好不过了。通常来说，我对嵌套函数还是心存警惕的，因为很容易在里面编写一些私有数据，并且在函数之间共享，这可能会增加代码的阅读和重构难度。

### 范例：在类之间搬移函数

在类之间搬移函数也是一种常见场景，下面我将用一个表示“账户”的 Account 类来讲解。

#### class Account...

```js
  get bankCharge() {
 let result = 4.5;
 if (this._daysOverdrawn > 0) result += this.overdraftCharge;
 return result;
}

get overdraftCharge() {
 if (this.type.isPremium) {
  const baseCharge = 10;
  if (this.daysOverdrawn <= 7)
   return baseCharge;
  else
   return baseCharge + (this.daysOverdrawn - 7) * 0.85;
 }
 else
  return this.daysOverdrawn * 1.75;
}
```

上面的代码会根据账户类型（account type）的不同，决定不同的“透支金额计费”算法。因此，很自然会想到将 overdraftCharge 函数搬移到 AccountType 类去。

第一步要做的是：观察被 overdraftCharge 使用的每一项特性，考虑是否值得将它们与 overdraftCharge 函数一起移动。此例中我需要让 daysOverdrawn 字段留在 Account 类中，因为它会随不同种类的账户而变化。

然后，我将 overdraftCharge 函数主体复制到 AccountType 类中，并做相应调整。

#### class AccountType...

```js
  overdraftCharge(daysOverdrawn) {
 if (this.isPremium) {
  const baseCharge  =  10;
  if (daysOverdrawn <= 7)
   return baseCharge;
  else
   return baseCharge + (daysOverdrawn - 7) * 0.85;
 }
 else
  return daysOverdrawn * 1.75;
}
```

为了使函数适应这个新家，我必须决定如何处理两个作用范围发生改变的变量。isPremium 如今只需要简单地从 this 上获取，但 daysOverdrawn 怎么办呢？我是直接传值，还是把整个 account 对象传过来？为了方便，我选择先简单传一个值，不过如果后续还需要账户（account）对象上除了 daysOverdrawn 以外的更多数据，例如需要根据账户类型（account type）来决定如何从账户（account）对象上取用数据时，那么我很可能会改变主意，转而选择传入整个 account 对象。

完成函数复制后，我会将原来的方法代之以一个委托调用。

#### class Account...

```js
  get bankCharge() {
 let result = 4.5;
 if (this._daysOverdrawn > 0) result += this.overdraftCharge;
 return result;
}

get overdraftCharge() {
 return this.type.overdraftCharge(this.daysOverdrawn);
}
```

然后下一件需要决定的事情是，是保留 overdraftCharge 这个委托函数，还是直接内联它？内联的话，代码会变成下面这样。

#### class Account...

```js
  get bankCharge() {
 let result = 4.5;
 if (this._daysOverdrawn > 0)
  result += this.type.overdraftCharge(this.daysOverdrawn);
 return result;
}
```

在早先的步骤中，我将 daysOverdrawn 作为参数直接传递给 overdraftCharge 函数，但如若账户（account）对象上有很多数据需要传递，那我就比较倾向于直接将整个对象作为参数传递过去：

#### class Account...

```js
  get bankCharge() {
 let result = 4.5;
 if (this._daysOverdrawn > 0) result += this.overdraftCharge;
 return result;
}

get overdraftCharge() {
 return this.type.overdraftCharge(this);
}
```

class AccountType…

```js
  overdraftCharge(account) {
 if (this.isPremium) {
  const baseCharge = 10;
  if (account.daysOverdrawn <= 7)
   return baseCharge;
  else
   return baseCharge + (account.daysOverdrawn - 7) * 0.85;
 }
 else
  return account.daysOverdrawn * 1.75;
}
```

## 8.2 搬移字段（Move Field）

```js
  class Customer {
  get plan() {return this._plan;}
  get discountRate() {return this._discountRate;}


  class Customer {
  get plan() {return this._plan;}
  get discountRate() {return this.plan.discountRate;}
```

### 动机

编程活动中你需要编写许多代码，为系统实现特定的行为，但往往数据结构才是一个健壮程序的根基。一个适应于问题域的良好数据结构，可以让行为代码变得简单明了，而一个糟糕的数据结构则将招致许多无用代码，这些代码更多是在差劲的数据结构中间纠缠不清，而非为系统实现有用的行为。代码凌乱，势必难以理解；不仅如此，坏的数据结构本身也会掩藏程序的真实意图。

因此，好的数据结构至关重要——不过这也与编程活动的许多方面一样，它们都很难一次做对。我通常都会做些预先的设计，设法得到最恰当的数据结构，此时如果你具备一些领域驱动设计（domain-driven design）方面的经验和知识，往往有助于你更好地设计数据结构。但即便经验再丰富，技能再熟练，我仍然发现我在进行初版设计时往往还是会犯错。在不断编程的过程中，我对问题域的理解会加深，对“什么是理想的数据结构”会有更多想法。这个星期看来合理而正确的设计决策，到了下个星期可能就不再正确了。

如果我发现数据结构已经不适应于需求，就应该马上修缮它。如果容许瑕疵存在并进一步累积，它们就会经常使我困惑，并且使代码愈来愈复杂。

我开始寻思搬移数据，可能是因为我发现每当调用某个函数时，除了传入一个记录参数，还总是需要同时传入另一条记录的某个字段一起作为参数。总是一同出现、一同作为函数参数传递的数据，最好是规整到同一条记录中，以体现它们之间的联系。修改的难度也是引起我注意的一个原因，如果修改一条记录时，总是需要同时改动另一条记录，那么说明很可能有字段放错了位置。此外，如果我更新一个字段时，需要同时在多个结构中做出修改，那也是一个征兆，表明该字段需要被搬移到一个集中的地点，这样每次只需修改一处地方。

搬移字段的操作通常是在其他更大的改动背景下发生的。实施字段搬移后，我可能会发现字段的诸多使用者应该通过目标对象来访问它，而不应该再通过源对象来访问。诸如此类的清理，我会在此后的重构中一并完成。同样，我也可能因为字段当前的一些用法而无法直接搬移它。我得先对其使用方式做一些重构，然后才能继续搬移工作。

到目前为止，我用以指称数据结构的术语都是“记录”（record），但以上论述对类和对象同样适用。类只是一种多了实例函数的记录，它与其他任何数据结构一样，都需要保持健康。不过类的实例函数确实简化了搬移数据的操作，因为它已经将数据的存取封装到访问函数中。当我搬移数据时，只需要相应修改访问函数的引用，该字段的所有客户端依然可以正常工作。因此，如果你的数据已经用类进行了封装，那么这个重构手法会更容易进行，我下面的展开也做了“通过类封装的数据更容易搬移”这个假设。如果你要搬移的数据是裸记录，没有任何封装，虽然类似的搬移仍然能够进行，但情况就会复杂一些。

### 做法

确保源字段已经得到了良好封装。

测试。

在目标对象上创建一个字段（及对应的访问函数）。

执行静态检查。

确保源对象里能够正常引用目标对象。

也许你已经有现成的字段或方法得到目标对象。如果没有，看看是否能简单地创建一个方法完成此事。如果还是不行，你可能就得在源对象里创建一个字段，用于存储目标对象了。这次修改可能留存很久，但你也可以只做临时修改，等到系统其他部分的重构完成就回来移除它。

调整源对象的访问函数，令其使用目标对象的字段。

如果源类的所有实例对象都共享对目标对象的访问权，那么可以考虑先更新源类的设值函数，让它修改源字段时，对目标对象上的字段做同样的修改。然后，再通过引入断言（302），当检测到源字段与目标字段不一致时抛出错误。一旦你确定改动没有引入任何可观察的行为变化，就可以放心地让访问函数直接使用目标对象的字段了。

测试。

移除源对象上的字段。

测试。

### 范例

我将用下面这个例子来介绍这项手法，其中 Customer 类代表了一位“顾客”，CustomerContract 代表与顾客关联的一个“合同”。

#### class Customer...

```js
  constructor(name, discountRate) {
 this._name = name;
 this._discountRate = discountRate;
 this._contract = new CustomerContract(dateToday());
}
get discountRate() {return this._discountRate;}
becomePreferred() {
 this._discountRate += 0.03;
 // other nice things
}
applyDiscount(amount) {
 return amount.subtract(amount.multiply(this._discountRate));
}
```

#### class CustomerContract...

```js
  constructor(startDate) {
  this._startDate = startDate;
}
```

我想要将折扣率（discountRate）字段从 Customer 类中搬移到 CustomerContract 里中。

第一件事情是先用封装变量（132）将对\_discountRate 字段的访问封装起来。

#### class Customer...

```js
  constructor(name, discountRate) {
 this._name = name;
 this._setDiscountRate(discountRate);
 this._contract = new CustomerContract(dateToday());
}
get discountRate() {return this._discountRate;}
_setDiscountRate(aNumber) {this._discountRate = aNumber;}
becomePreferred() {
 this._setDiscountRate(this.discountRate + 0.03);
 // other nice things
}
applyDiscount(amount)  {
 return amount.subtract(amount.multiply(this.discountRate));
}
```

我通过定制的 applyDiscount 方法来更新字段，而不是使用通常的设值函数，这是因为我不想为字段暴露一个 public 的设值函数。

接着我在 CustomerContract 中添加一个对应的字段和访问函数。

#### class CustomerContract...

```js
  constructor(startDate, discountRate) {
 this._startDate = startDate;
 this._discountRate = discountRate;
}
get discountRate()  {return this._discountRate;}
set discountRate(arg) {this._discountRate = arg;}
```

接下来，我可以修改 customer 对象的访问函数，让它引用 CustomerContract 这个新添的字段。不过当我这么干时，我收到了一个错误：“Cannot set property 'discountRate' of undefined”。这是因为我们先调用了 `_setDiscountRate` 函数，而此时 CustomerContract 对象尚未创建出来。为了修复这个错误，我得先撤销刚刚的代码，回到上一个可工作的状态，然后再应用移动语句（223）手法，将`_setDiscountRate` 函数调用语句挪动到创建对象的语句之后。

#### class Customer...

```js
  constructor(name, discountRate) {
  this._name = name;
  this._setDiscountRate(discountRate);
  this._contract = new CustomerContract(dateToday());
}
```

搬移完语句后运行一下测试。测试通过后，再次修改 Customer 的访问函数，让它使用\_contract 对象上的 discountRate 字段。

#### class Customer...

```js
  get discountRate() {return this._contract.discountRate;}
_setDiscountRate(aNumber) {this._contract.discountRate = aNumber;}
```

在 JavaScript 中，使用类的字段无须事先声明，因此替换完访问函数，实际上已经没有其他字段再需要我删除。

搬移裸记录

搬移字段这项重构手法对于类的实例对象通常较易进行，因为将数据访问包装到方法中，是类所天然支持的一种封装手段。如果我要搬移的字段是裸记录，并且被许多函数直接访问，那么这项重构仍然很有意义，只不过情况会复杂不少。

我可以先为记录创建相应的访问函数，并修改所有读取和更新记录的地方，使它们引用新创建的访问函数。如果待搬移的字段是不可变（immutable）的，那么我可以在设值函数中同时更新源字段和目标字段，然后再逐步迁移读取记录的调用点。不过，我依然会尽可能先用封装记录（162）手法将记录封装成类，如此一来后续修改会更加简单。

### 范例：搬移字段到共享对象

现在，让我们看另外一个场景。还是那个代表“账户”的 Account 类，类上有一个代表“利率”的字段\_interestRate。

#### class Account...

```js
constructor(number, type, interestRate) {
 this._number = number;
 this._type = type;
 this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

#### class AccountType...

```js
constructor(nameString) {
  this._name = nameString;
}
```

我不希望让每个账户自己维护一个利率字段，利率应该取决于账户本身的类型，因此我想将它搬移到 AccountType 中去。

利率字段已经通过访问函数得到了良好的封装，因此我只需要在 AccountType 上创建对应的字段及访问函数即可。

#### class AccountType...

```js
constructor(nameString, interestRate) {
  this._name = nameString;
  this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

接着我应该着手替换 Account 类的访问函数，但我发现直接替换可能有个潜藏的问题。在重构之前，每个账户都自己维护一份利率数据，而现在我要让所有相同类型的账户共享同一个利率值。如果当前类型相同的账户确实拥有相同的利率，那么这次重构就能成立，因为这不会引起可观测的行为变化。但只要存在一个特例，即同一类型的账户可能有不同的利率值，那么这样的修改就不叫重构了，因为它会改变系统的可观测行为。倘若账户的数据保存在数据库中，那我就应该检查一下数据库，确保同一类型的账户都拥有与其账户类型匹配的利率值。同时，我还可以在 Account 类引入断言（302），确保出现异常的利率数据时能够及时发现。

#### class Account...

```js
constructor(number, type, interestRate) {
  this._number = number;
  this._type = type;
  assert(interestRate === this._type.interestRate);
  this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

我会保留这条断言，让系统先运行一段时间，看看是否会在这捕获到错误。或者，除了添加断言，我还可以将错误记录到日志里。一段时间后，一旦我对代码变得更加自信，确定它确实没有引起行为变化后，我就可以让 Account 直接访问 AccountType 上的 interestRate 字段，并将原来的字段完全删除了。

#### class Account...

```js
constructor(number, type) {
  this._number = number;
  this._type = type;
}
get interestRate() {return this._type.interestRate;}
```

## 8.3 搬移语句到函数（Move Statements into Function）

反向重构：搬移语句到调用者（217）

```js
result.push(`<p>title: ${person.photo.title}</p>`);
result.concat(photoData(person.photo));

function photoData(aPhoto) {
  return [
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ];
}

result.concat(photoData(person.photo));

function photoData(aPhoto) {
  return [
    `<p>title: ${aPhoto.title}</p>`,
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ];
}
```

### 动机

要维护代码库的健康发展，需要遵守几条黄金守则，其中最重要的一条当属“消除重复”。如果我发现调用某个函数时，总有一些相同的代码也需要每次执行，那么我会考虑将此段代码合并到函数里头。这样，日后对这段代码的修改只需改一处地方，还能对所有调用者同时生效。如果将来代码对不同的调用者需有不同的行为，那时再通过搬移语句到调用者（217）将它（或其一部分）搬移出来也十分简单。

如果某些语句与一个函数放在一起更像一个整体，并且更有助于理解，那我就会毫不犹豫地将语句搬移到函数里去。如果它们与函数不像一个整体，但仍应与函数一起执行，那我可以用提炼函数（106）将语句和函数一并提炼出去。这基本就是我下面要描述的做法了，只是下面还多了内联和改名的步骤。这些清理工作通常有其必要性，可以在完成核心步骤后再择机完成。

### 做法

如果重复的代码段离调用目标函数的地方还有些距离，则先用移动语句（223）将这些语句挪动到紧邻目标函数的位置。

如果目标函数仅被唯一一个源函数调用，那么只需将源函数中的重复代码段剪切并粘贴到目标函数中即可，然后运行测试。本做法的后续步骤至此可以忽略。

如果函数不止一个调用点，那么先选择其中一个调用点应用提炼函数（106），将待搬移的语句与目标函数一起提炼成一个新函数。给新函数取个临时的名字，只要易于搜索即可。

调整函数的其他调用点，令它们调用新提炼的函数。每次调整之后运行测试。

完成所有引用点的替换后，应用内联函数（115）将目标函数内联到新函数里，并移除原目标函数。

对新函数应用函数改名（124），将其改名为原目标函数的名字。

如果你能想到更好的名字，那就用更好的那个。

### 范例

我将用一个例子来讲解这项手法。以下代码会生成一些关于相片（photo）的 HTML：

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(`<p>title: ${person.photo.title}</p>`);
  result.push(emitPhotoData(person.photo));
  return result.join("\n");
}
function photoDiv(p) {
  return [
    "<div>",
    `<p>title:  ${p.title}</p>`,
    emitPhotoData(p),
    "</div>",
  ].join("\n");
}

function emitPhotoData(aPhoto) {
  const result = [];
  result.push(`<p>location: ${aPhoto.location}</p>`);
  result.push(`<p>date: ${aPhoto.date.toDateString()}</p>`);
  return result.join("\n");
}
```

这个例子中的 emitPhotoData 函数有两个调用点，每个调用点的前面都有一行类似的重复代码，用于打印与标题（title）相关的信息。我希望能消除重复，把打印标题的那行代码搬移到 emitPhotoData 函数里去。如果 emitPhotoData 只有一个调用点，那我大可直接把代码复制并粘贴过去就完事，但若调用点不止一个，那我就更倾向于用更安全的手法小步前进。

我先选择其中一个调用点，对其应用提炼函数（106）。除了我想搬移的语句，我还把 emitPhotoData 函数也一起提炼到新函数中。

```js
function photoDiv(p) {
  return ["<div>", zznew(p), "</div>"].join("\n");
}

function zznew(p) {
  return [`<p>title: ${p.title}</p>`, emitPhotoData(p)].join("\n");
}
```

完成提炼后，我会逐一查看 emitPhotoData 的其他调用点，找到该函数与其前面的重复语句，一并换成对新函数的调用。

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(zznew(person.photo));
  return result.join("\n");
}
```

替换完 emitPhotoData 函数的所有调用点后，我紧接着应用内联函数（115）将 emitPhotoData 函数内联到新函数中。

```js
function zznew(p) {
  return [
    `<p>title: ${p.title}</p>`,
    `<p>location: ${p.location}</p>`,
    `<p>date: ${p.date.toDateString()}</p>`,
  ].join("\n");
}
```

最后，再对新提炼的函数应用函数改名（124），就大功告成了。

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(emitPhotoData(person.photo));
  return result.join("\n");
}

function photoDiv(aPhoto) {
  return ["<div>", emitPhotoData(aPhoto), "</div>"].join("\n");
}

function emitPhotoData(aPhoto) {
  return [
    `<p>title: ${aPhoto.title}</p>`,
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ].join("\n");
}
```

同时我会记得调整函数参数的命名，使之与我的编程风格保持一致。

## 8.4 搬移语句到调用者（Move Statements to Callers）

反向重构：搬移语句到函数（213）

```js
emitPhotoData(outStream, person.photo);

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>location: ${photo.location}</p>\n`);
}

emitPhotoData(outStream, person.photo);
outStream.write(`<p>location: ${person.photo.location}</p>\n`);

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
}
```

### 动机

作为程序员，我们的职责就是设计出结构一致、抽象合宜的程序，而程序抽象能力的源泉正是来自函数。与其他抽象机制的设计一样，我们并非总能平衡好抽象的边界。随着系统能力发生演进（通常只要是有用的系统，功能都会演进），原先设定的抽象边界总会悄无声息地发生偏移。对于函数来说，这样的边界偏移意味着曾经视为一个整体、一个单元的行为，如今可能已经分化出两个甚至是多个不同的关注点。

函数边界发生偏移的一个征兆是，以往在多个地方共用的行为，如今需要在某些调用点面前表现出不同的行为。于是，我们得把表现不同的行为从函数里挪出，并搬移到其调用处。这种情况下，我会使用移动语句（223）手法，先将表现不同的行为调整到函数的开头或结尾，再使用本手法将语句搬移到其调用点。只要差异代码被搬移到调用点，我就可以根据需要对其进行修改。

这个重构手法比较适合处理边界仅有些许偏移的场景，但有时调用点和调用者之间的边界已经相去甚远，此时便只能重新进行设计了。若果真如此，最好的办法是先用内联函数（115）合并双方的内容，调整语句的顺序，再提炼出新的函数来，以形成更合适的边界。

### 做法

最简单的情况下，原函数非常简单，其调用者也只有寥寥一两个，此时只需把要搬移的代码从函数里剪切出来并粘贴回调用端去即可，必要的时候做些调整。运行测试。如果测试通过，那就大功告成，本手法可以到此为止。

若调用点不止一两个，则需要先用提炼函数（106）将你不想搬移的代码提炼成一个新函数，函数名可以临时起一个，只要后续容易搜索即可。

如果原函数是一个超类方法，并且有子类进行了覆写，那么还需要对所有子类的覆写方法进行同样的提炼操作，保证继承体系上每个类都有一份与超类相同的提炼函数。接着将子类的提炼函数删除，让它们引用超类提炼出来的函数。

对原函数应用内联函数（115）。

对提炼出来的函数应用改变函数声明（124），令其与原函数使用同一个名字。

如果你能想到更好的名字，那就用更好的那个。

### 范例

下面这个例子比较简单：emitPhotoData 是一个函数，在两处地方被调用。

```js
  function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 emitPhotoData(outStream, person.photo);
}

function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   emitPhotoData(outStream, p);
   outStream.write("</div>\n");
  });
}

function emitPhotoData(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
 outStream.write(`<p>location: ${photo.location}</p>\n`);
}
```

我需要修改软件，支持 listRecentPhotos 函数以不同方式渲染相片的 location 信息，而 renderPerson 的行为则保持不变。为了使这次修改更容易进行，我要应用本手法，将 emitPhotoData 函数最后的那行代码搬移到其调用端。

一般来说，像这样简单的场景，我都会直接将 emitPhotoData 的最后一行剪切并粘贴到两个调用它的函数后面。但为了演示这项重构手法如何在更复杂的场景下运作，这里我还是遵从更详细也更安全的步骤。

重构的第一步是先用提炼函数（106），将那些最终希望留在 emitPhotoData 函数里的语句先提炼出去。

```js
  function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 emitPhotoData(outStream, person.photo);
}

function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   emitPhotoData(outStream, p);
   outStream.write("</div>\n");
  });
}

function  emitPhotoData(outStream, photo) {
 zztmp(outStream,  photo);
 outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

新提炼出来的函数一般只会短暂存在，因此我在命名上不需要太认真，不过，取个容易搜索的名字会很有帮助。提炼完成后运行一下测试，确保提炼出来的新函数能正常工作。

接下来，我要对 emitPhotoData 的调用点逐一应用内联函数（115）。先从 renderPerson 函数开始。

```js
  function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 zztmp(outStream,  person.photo);
 outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}
function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   emitPhotoData(outStream, p);
   outStream.write("</div>\n");
  });
}

function emitPhotoData(outStream, photo) {
 zztmp(outStream, photo);
 outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

然后再次运行测试，确保这次函数内联能正常工作。测试通过后，再前往下一个调用点。

```js
  function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 zztmp(outStream,  person.photo);
 outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   zztmp(outStream, p);
   outStream.write(`<p>location: ${p.location}</p>\n`);
   outStream.write("</div>\n");
  });
}

function emitPhotoData(outStream, photo) {
 zztmp(outStream, photo);
 outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

至此，我就可以移除外面的 emitPhotoData 函数，完成内联函数（115）手法。

```js
function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 zztmp(outStream,  person.photo);
 outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   zztmp(outStream, p);
   outStream.write(`<p>location: ${p.location}</p>\n`);
   outStream.write("</div>\n");
  });
}

function emitPhotoData(outStream, photo) {
 zztmp(outStream, photo);
 outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

最后，我将 zztmp 改名为原函数的名字 emitPhotoData，完成本次重构。

```js
function renderPerson(outStream, person) {
 outStream.write(`<p>${person.name}</p>\n`);
 renderPhoto(outStream, person.photo);
 emitPhotoData(outStream, person.photo);
 outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
 photos
  .filter(p => p.date > recentDateCutoff())
  .forEach(p => {
   outStream.write("<div>\n");
   emitPhotoData(outStream, p);
   outStream.write(`<p>location: ${p.location}</p>\n`);
   outStream.write("</div>\n");
  });
}

function emitPhotoData(outStream, photo) {
 outStream.write(`<p>title: ${photo.title}</p>\n`);
 outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

## 8.5 以函数调用取代内联代码（Replace Inline Code with Function Call）

```js
let appliesToMass = false;
for (const s of states) {
  if (s === "MA") appliesToMass = true;
}

appliesToMass = states.includes("MA");
```

### 动机

善用函数可以帮助我将相关的行为打包起来，这对于提升代码的表达力大有裨益—— 一个命名良好的函数，本身就能极好地解释代码的用途，使读者不必了解其细节。函数同样有助于消除重复，因为同一段代码我不需要编写两次，每次调用一下函数即可。此外，当我需要修改函数的内部实现时，也不需要四处寻找有没有漏改的相似代码。（当然，我可能需要检查函数的所有调用点，判断它们是否都应该使用新的实现，但通常很少需要这么仔细，即便需要，也总好过四处寻找相似代码。）

如果我见到一些内联代码，它们做的事情仅仅是已有函数的重复，我通常会以一个函数调用取代内联代码。但有一种情况需要特殊对待，那就是当内联代码与函数之间只是外表相似但其实并无本质联系时。这种情况下，当我改变了函数实现时，并不期望对应内联代码的行为发生改变。判断内联代码与函数之间是否真正重复，从函数名往往可以看出端倪：如果一个函数命名得当，也确实与内联代码做了一样的事，那么这个名字用在内联代码的语境里也应该十分协调；如果函数名显得不协调，可能是因为命名本身就比较糟糕（此时可以运用函数改名（124）来解决），也可能是因为函数与内联代码彼此的用途确实有所不同。若是后者的情况，我就不应该用函数调用取代该内联代码。

我发现，配合一些库函数使用，会使本手法效果更佳，因为我甚至连函数体都不需要自己编写了，库已经提供了相应的函数。

### 做法

将内联代码替代为对一个既有函数的调用。

测试。

## 8.6 移动语句（Slide Statements）

曾用名：合并重复的代码片段（Consolidate Duplicate Conditional Fragments）

```js
const pricingPlan = retrievePricingPlan();
const order = retreiveOrder();
let charge;
const chargePerUnit = pricingPlan.unit;

const pricingPlan = retrievePricingPlan();
const chargePerUnit = pricingPlan.unit;
const order = retreiveOrder();
let charge;
```

### 动机

让存在关联的东西一起出现，可以使代码更容易理解。如果有几行代码取用了同一个数据结构，那么最好是让它们在一起出现，而不是夹杂在取用其他数据结构的代码中间。最简单的情况下，我只需使用移动语句就可以让它们聚集起来。此外还有一种常见的“关联”，就是关于变量的声明和使用。有人喜欢在函数顶部一口气声明函数用到的所有变量，我个人则喜欢在第一次需要使用变量的地方再声明它。

通常来说，把相关代码搜集到一处，往往是另一项重构（通常是在提炼函数（106））开始之前的准备工作。相比于仅仅把几行相关的代码移动到一起，将它们提炼到独立的函数往往能起到更好的抽象效果。但如果起先存在关联的代码就没有彼此在一起，那么我也很难应用提炼函数（106）的手法。

### 做法

确定待移动的代码片段应该被搬往何处。仔细检查待移动片段与目的地之间的语句，看看搬移后是否会影响这些代码正常工作。如果会，则放弃这项重构。

往前移动代码片段时，如果片段中声明了变量，则不允许移动到任何变量的声明语句之前。往后移动代码片段时，如果有语句引用了待移动片段中的变量，则不允许移动到该语句之后。往后移动代码片段时，如果有语句修改了待移动片段中引用的变量，则不允许移动到该语句之后。往后移动代码片段时，如果片段中修改了某些元素，则不允许移动到任何引用了这些元素的语句之后。

剪切源代码片段，粘贴到上一步选定的位置上。

测试。

如果测试失败，那么尝试减小移动的步子：要么是减少上下移动的行数，要么是一次搬移更少的代码。

### 范例

移动代码片段时，通常需要想清楚两件事：本次调整的目标是什么，以及该目标能否达到。第一件事通常取决于代码所在的上下文。最简单的情况是，我希望元素的声明点和使用点互相靠近，因此移动语句的目标便是将元素的声明语句移动到靠近它们的使用处。不过大多数时候，我移动代码的动机都是因为想做另一项重构，比如在应用提炼函数（106）之前先将相关的代码集中到一块，以方便做函数提炼。

确定要把代码移动到哪里之后，我就需要思考第二个问题，也就是此次搬移能否做到的问题。为此我需要观察待移动的代码，以及移动中间经过的代码段，我得思考这个问题：如果我把代码移动过去，执行次序的不同会不会使代码之间产生干扰，甚至于改变程序的可观测行为？

请观察以下代码片段：

```js
 1 const pricingPlan = retrievePricingPlan();
 2 const order = retreiveOrder();
 3 const baseCharge = pricingPlan.base;
 4 let charge;
 5 const chargePerUnit = pricingPlan.unit;
 6 const units = order.units;
 7 let discount;
 8 charge = baseCharge + units * chargePerUnit;
 9 let discountableUnits = Math.max(units - pricingPlan.discountThreshold, 0);
10 discount = discountableUnits * pricingPlan.discountFactor;
11 if (order.isRepeat) discount += 20;
12 charge = charge - discount;
13 chargeOrder(charge);
```

前七行是变量的声明语句，移动它们通常很简单。假如我想把与处理折扣（discount）相关的代码搬移到一起，那么我可以直接将第 7 行（let discount）移动到第 10 行上面（discount = ...那一行）。因为变量声明没有副作用，也不会引用其他变量，所以我可以很安全地将声明语句往后移动，一直移动到引用 discount 变量的语句之上。此种类型的语句移动也十分常见——当我要提炼函数（106）时，通常得先将相关变量的声明语句搬移过来。

我会再寻找类似的没有副作用的变量声明语句。类似地，我可以毫无障碍地把声明了 order 变量的第 2 行（const order = ...）移动到使用它的第 6 行（const units = ...）上面。

上面搬移变量声明语句之所以顺利，除了因为语句本身没有副作用，还得益于我移动语句时跨过的代码片段同样没有副作用。事实上，对于没有副作用的代码，我几乎可以随心所欲地编排它们的顺序，这也是优秀的程序员都会尽量编写无副作用代码的原因之一。

当然，这里还有一个小细节，那就是我从何得知第 2 行代码没有副作用呢？我只有深入检查 retrieveOrder()函数的内部实现，才能真正确保它确实没有副作用（除了检查函数本身，还得检查它内部调用的函数都没有副作用，以及它调用的函数内部调用的函数都没有副作用……一直检查到调用链的底端）。实践中，我编写代码总是尽量遵循命令与查询分离（Command-Query Separation）[mf-cqs]原则，在这个前提下，我可以确定任何有返回值的函数都不存在副作用。但只有在我了解代码库的前提下才如此自信；如果我对代码库还不熟悉，我就得更加小心。但在我自己的编码过程中，我确实总是尽量遵循命令与查询分离的模式，因为它让我一眼就能看清代码有无副作用，而这件事情真是价值不菲。

如果待移动的代码片段本身有副作用，或者它需要跨越的代码存在副作用，移动它们时就必须加倍小心。我得仔细寻找两个代码片段中间的代码有没有副作用，是不是对执行次序敏感。因此，假设我想将第 11 行（if(order.isRepeat)...）挪动到段落底部，我会发现行不通，因为中间第 12 行语句引用了 discount 变量，而我在第 11 行中可能改动这个变量；类似地，假设我想将第 13 行（chargeOrder(charge)）往上搬移，那也是行不通的，因为第 13 行引用的 charge 变量在第 12 行会被修改。不过，如果我想将第 8 行代码（charge = baseCharge + ...）移动到第 9 行到第 11 行中间的任意地方则是可行的，因为这几行都未修改任何变量的状态。

移动代码时，最容易遵守的一条规则是，如果待移动代码片段中引用的变量在另一个代码片段中被修改了，那我就不能安全地将前者移动到后者之后；同样，如果前者会修改后者中引用的变量，也一样不能安全地进行上述移动。但这条规则仅仅作为参考，它也不是绝对的，比如下面这个例子，虽然两个语句都修改了彼此之间的变量，但我仍能安全地调整它们的先后顺序。

```js
a = a + 10;
a = a + 5;
```

但无论如何，要判断一次语句移动是否安全，都意味着我得真正理解代码的工作原理，以及运算符之间的组合方式等。

正因此项重构如此需要关注状态更新，所以我会尽量移除那些会更新元素状态的代码。比如此例中的 charge 变量，在移动其相关的代码之前，我会先看看是否能对它应用拆分变量（240）手法。

以上的分析都比较简单，没什么难度，因为代码里修改的都是局部变量，局部变量是比较好处理的。但处理更复杂的数据结构时，情况就不同了，判断代码段之间是否存在相互干扰会困难得多。这时测试扮演了重要角色：每次移动代码之后运行测试，看看有没有任何测试失败。如果我的测试覆盖足够全面，我就会对这次重构比较有信心；但如果测试覆盖不够，我就得小心一些了——通常，我会先改善代码的测试，然后再进行重构。

如果移动过后测试失败了，那么意味着我得减小移动的步子，比如一次先移动 5 行，而不是 10 行；或者先移动到那些看着比较可能出错的代码上面，但不越过它，看看效果。同时，测试失败也可能是一个征兆，提醒我这次移动可能还不是时候，可能还需要在别处先做一些其他的工作。

### 范例：包含条件逻辑的移动

对于拥有条件逻辑的代码，移动手法同样适用。当从条件分支中移走代码时，通常是要消除重复逻辑；将代码移入条件分支时，通常是反过来，有意添加一些重复逻辑。

在下面这个例子中，两个条件分支里都有一个相同的语句：

```js
let result;
if (availableResources.length === 0) {
  result = createResource();
  allocatedResources.push(result);
} else {
  result = availableResources.pop();
  allocatedResources.push(result);
}
return result;
```

我可以将这两句重复代码从条件分支中移走，只在 if-else 块的末尾保留一句。

```js
let result;
if (availableResources.length === 0) {
  result = createResource();
} else {
  result = availableResources.pop();
}
allocatedResources.push(result);
return result;
```

这个手法同样可以反过来用，也就是把一个语句分别搬移到不同的条件分支里，这样会在每个条件分支里留下同一段重复的代码。

延伸阅读

除了我介绍的这个方法，我还见过一个十分相似的重构手法，名字叫作“交换语句位置”（Swap Statement）[wake-swap]。该手法同样适用于移动相邻的代码片段，只不过它适用的是只有一条语句的片段。你可以把它想成移动语句手法的一个特例，也就是待移动的代码片段以及它所跨过的代码片段，都只有一条语句。我对这项重构手法很感兴趣，毕竟我也一直在强调小步修改——有时甚至小步到于初学重构的人看来都很不可思议的地步。

但最后，我还是选择在本重构手法中介绍如何移动范围更大的代码片段，因为我自己平时就是这么做的。我只有在处理大范围的语句移动遇到困难时才会变得小步、一次只移动一条语句，但即便是这样的困难我也很少遇见。无论如何，当代码过于复杂凌乱时，小步的移动通常会更加顺利。

## 8.7 拆分循环（Split Loop）

```js
let averageAge = 0;
let totalSalary = 0;
for (const p of people) {
  averageAge += p.age;
  totalSalary += p.salary;
}
averageAge = averageAge / people.length;

let totalSalary = 0;
for (const p of people) {
  totalSalary += p.salary;
}

let averageAge = 0;
for (const p of people) {
  averageAge += p.age;
}
averageAge = averageAge / people.length;
```

### 动机

你常常能见到一些身兼多职的循环，它们一次做了两三件事情，不为别的，就因为这样可以只循环一次。但如果你在一次循环中做了两件不同的事，那么每当需要修改循环时，你都得同时理解这两件事情。如果能够将循环拆分，让一个循环只做一件事情，那就能确保每次修改时你只需要理解要修改的那块代码的行为就可以了。

拆分循环还能让每个循环更容易使用。如果一个循环只计算一个值，那么它直接返回该值即可；但如果循环做了太多件事，那就只得返回结构型数据或者通过局部变量传值了。因此，一般拆分循环后，我还会紧接着对拆分得到的循环应用提炼函数（106）。

这项重构手法可能让许多程序员感到不安，因为它会迫使你执行两次循环。对此，我一贯的建议也与 2.8 节里所明确指出的一致：先进行重构，然后再进行性能优化。我得先让代码结构变得清晰，才能做进一步优化；如果重构之后该循环确实成了性能的瓶颈，届时再把拆开的循环合到一起也很容易。但实际情况是，即使处理的列表数据更多一些，循环本身也很少成为性能瓶颈，更何况拆分出循环来通常还使一些更强大的优化手段变得可能。

### 做法

复制一遍循环代码。

识别并移除循环中的重复代码，使每个循环只做一件事。

测试。

完成循环拆分后，考虑对得到的每个循环应用提炼函数（106）。

### 范例

下面我以一段循环代码开始。该循环会计算需要支付给所有员工的总薪水（total salary），并计算出最年轻（youngest）员工的年龄。

```js
let youngest = people[0] ? people[0].age : Infinity;
let totalSalary = 0;
for (const p of people) {
 if (p.age < youngest) youngest = p.age;
 totalSalary += p.salary;
}

return `youngestAge: ${youngest}, totalSalary: ${totalSalary}`;
```

该循环十分简单，但仍然做了两种不同的计算。要拆分这两种计算，我要先复制一遍循环代码。

```js
let youngest = people[0] ? people[0].age : Infinity;
let totalSalary = 0;
for (const p of people) {
 if (p.age < youngest) youngest = p.age;
 totalSalary += p.salary;
}
for (const p of people) {
 if (p.age < youngest) youngest = p.age;
 totalSalary += p.salary;
}

return `youngestAge: ${youngest}, totalSalary: ${totalSalary}`;
```

复制过后，我需要将循环中重复的计算逻辑删除，否则就会累加出错误的结果。如果循环中的代码没有副作用，那便可以先留着它们不删除，可惜上述例子并不符合这种情况。

```js
let youngest = people[0] ? people[0].age : Infinity;
let totalSalary = 0;
for (const p of people) {
 if (p.age < youngest) youngest = p.age;
 totalSalary += p.salary;
}

for (const p of people) {
 if (p.age < youngest) youngest = p.age;
 totalSalary += p.salary;
}

return `youngestAge: ${youngest}, totalSalary: ${totalSalary}`;
```

至此，拆分循环这个手法本身的内容就结束了。但本手法的意义不仅在于拆分出循环本身，而且在于它为进一步优化提供了良好的起点——下一步我通常会寻求将每个循环提炼到独立的函数中。在做提炼之前，我得先用移动语句（223）微调一下代码顺序，将与循环相关的变量先搬移到一起：

```js
let totalSalary = 0;
for (const p of people) {
 totalSalary += p.salary;
}

let youngest = people[0] ? people[0].age : Infinity;
for (const p of people) {
 if (p.age < youngest) youngest = p.age;
}

return `youngestAge: ${youngest}, totalSalary: ${totalSalary}`;
```

然后，我就可以顺利地应用提炼函数（106）了。

```js
  return `youngestAge: ${youngestAge()}, totalSalary: ${totalSalary()}`;

function totalSalary() {
 let totalSalary = 0;
 for (const p of people) {
  totalSalary += p.salary;
 }
 return totalSalary;
}

function youngestAge() {
 let youngest = people[0] ? people[0].age : Infinity;
 for (const p of people) {
  if (p.age < youngest) youngest = p.age;
 }
 return youngest;
}
```

对于像 totalSalary 这样的累加计算，我绝少能抵挡得住进一步使用以管道取代循环（231）重构它的诱惑；而对于 youngestAge 的计算，显然可以用替换算法（195）替之以更好的算法。

```js
  return `youngestAge: ${youngestAge()}, totalSalary: ${totalSalary()}`;

function totalSalary() {
 return people.reduce((total,p) => total + p.salary, 0);
}
function youngestAge() {
 return Math.min(...people.map(p => p.age));
}
```

## 8.8 以管道取代循环（Replace Loop with Pipeline）

```js
const names = [];
for (const i of input) {
  if (i.job === "programmer")
    names.push(i.name);
}


const names = input
  .filter(i => i.job === "programmer")
  .map(i => i.name)
;
```

### 动机

与大多数程序员一样，我入行的时候也有人告诉我，迭代一组集合时得使用循环。不过时代在发展，如今越来越多的编程语言都提供了更好的语言结构来处理迭代过程，这种结构就叫作集合管道（collection pipeline）。集合管道[mf-cp]是这样一种技术，它允许我使用一组运算来描述集合的迭代过程，其中每种运算接收的入参和返回值都是一个集合。这类运算有很多种，最常见的则非 map 和 filter 莫属：map 运算是指用一个函数作用于输入集合的每一个元素上，将集合变换成另外一个集合的过程；filter 运算是指用一个函数从输入集合中筛选出符合条件的元素子集的过程。运算得到的集合可以供管道的后续流程使用。我发现一些逻辑如果采用集合管道来编写，代码的可读性会更强——我只消从头到尾阅读一遍代码，就能弄清对象在管道中间的变换过程。

### 做法

创建一个新变量，用以存放参与循环过程的集合。

也可以简单地复制一个现有的变量赋值给新变量。

从循环顶部开始，将循环里的每一块行为依次搬移出来，在上一步创建的集合变量上用一种管道运算替代之。每次修改后运行测试。

搬移完循环里的全部行为后，将循环整个删除。

如果循环内部通过累加变量来保存结果，那么移除循环后，将管道运算的最终结果赋值给该累加变量。

### 范例

在这个例子中，我们有一个 CSV 文件，里面存有各个办公室（office）的一些数据。

```
office, country, telephone
Chicago, USA, +1 312 373 1000
Beijing, China, +86 4008 900 505
Bangalore, India, +91 80 4064 9570
Porto Alegre, Brazil, +55 51 3079 3550
Chennai, India, +91 44 660 44766

... (more data follows)
```

下面这个 acquireData 函数的作用是从数据中筛选出印度的所有办公室，并返回办公室所在的城市（city）信息和联系电话（telephone number）。

```js
function acquireData(input) {
  const lines = input.split("\n");
  let firstLine = true;
  const result = [];
  for (const line of lines) {
    if (firstLine) {
      firstLine = false;
      continue;
    }
    if (line.trim() === "") continue;
    const record = line.split(",");
    if (record[1].trim() === "India") {
      result.push({ city: record[0].trim(), phone: record[2].trim() });
    }
  }
  return result;
}
```

这个循环略显复杂，我希望能用一组管道操作来替换它。

第一步是先创建一个独立的变量，用来存放参与循环过程的集合值。

```js
function acquireData(input) {
  const lines = input.split("\n");
  let firstLine = true;
  const result = [];
  const loopItems = lines;
  for (const line of loopItems) {
    if (firstLine) {
      firstLine = false;
      continue;
    }
    if (line.trim() === "") continue;
    const record = line.split(",");
    if (record[1].trim() === "India") {
      result.push({ city: record[0].trim(), phone: record[2].trim() });
    }
  }
  return result;
}
```

循环第一部分的作用是在忽略 CSV 文件的第一行数据。这其实是一个切片（slice）操作，因此我先从循环中移除这部分代码，并在集合变量的声明后面新增一个对应的 slice 运算来替代它。

```js
function acquireData(input) {
  const lines = input.split("\n");
  let firstLine = true;
  const result = [];
  const loopItems = lines.slice(1);
  for (const line of loopItems) {
    if (firstLine) {
      firstLine = false;
      continue;
    }
    if (line.trim() === "") continue;
    const record = line.split(",");
    if (record[1].trim() === "India") {
      result.push({ city: record[0].trim(), phone: record[2].trim() });
    }
  }
  return result;
}
```

从循环中删除代码还有一个好处，那就是 firstLine 这个控制变量也可以一并移除了——无论何时，删除控制变量总能使我身心愉悦。

该循环的下一个行为是要移除数据中的所有空行。这同样可用一个过滤（filter）运算替代之。

```js
function acquireData(input) {
 const lines = input.split("\n");
 const result = [];
 const loopItems = lines
    .slice(1)
    .filter(line => line.trim() !== "")
    ;
 for (const line of loopItems) {
  if (line.trim() === "") continue;
  const  record = line.split(",");
  if (record[1].trim() === "India") {
   result.push({city: record[0].trim(), phone: record[2].trim()});
  }
 }
 return result;
}
```

    编写管道运算时，我喜欢让结尾的分号单占一行，这样方便调整管道的结构。

接下来是将数据的一行转换成数组，这明显可以用一个 map 运算替代。然后我们还发现，原来的 record 命名其实有误导性，它没有体现出“转换得到的结果是数组”这个信息，不过既然现在还在做其他重构，先不动它会比较安全，回头再为它改名也不迟。

```js
function acquireData(input) {
 const lines = input.split("\n");
 const result = [];
 const loopItems = lines
    .slice(1)
    .filter(line => line.trim() !== "")
    .map(line => line.split(","))
    ;
 for (const line of loopItems) {
  const record = line;.split(",");
  if (record[1].trim() === "India") {
   result.push({city: record[0].trim(), phone: record[2].trim()});
  }
 }
 return result;
}
```

然后又是一个过滤（filter）操作，只从结果中筛选出印度办公室的记录。

```js
function acquireData(input) {
 const lines = input.split("\n");
 const result = [];
 const loopItems = lines
    .slice(1)
    .filter(line => line.trim() !== "")
    .map(line => line.split(","))
    .filter(record => record[1].trim() === "India")
    ;
 for (const line of loopItems) {
  const record = line;
  if (record[1].trim() === "India") {
   result.push({city: record[0].trim(), phone: record[2].trim()});
  }
 }
 return result;
}
```

最后再把结果映射（map）成需要的记录格式：

```js
  function acquireData(input) {
 const lines = input.split("\n");
 const result = [];
 const loopItems = lines
    .slice(1)
    .filter(line => line.trim() !== "")
    .map(line => line.split(","))
    .filter(record => record[1].trim() === "India")
    .map(record => ({city: record[0].trim(), phone: record[2].trim()}))
    ;
 for (const line of loopItems) {
  const record = line;
  result.push(line);
 }
 return result;
}
```

现在，循环剩余的唯一作用就是对累加变量赋值了。我可以将上面管道产出的结果赋值给该累加变量，然后删除整个循环：

```js
function acquireData(input) {
 const lines = input.split("\n");
 const result = lines
    .slice(1)
    .filter(line => line.trim() !== "")
    .map(line => line.split(","))
    .filter(record => record[1].trim() === "India")
    .map(record => ({city: record[0].trim(), phone: record[2].trim()}))
    ;
 for (const line of loopItems) {
  const record = line;
  result.push(line);
 }
 return result;
}
```

以上就是本手法的全部精髓所在了。不过后续还有些清理工作可做：我内联了 result 变量，为一些函数变量改名，最后还对代码进行布局，让它读起来更像个表格。

```js
function acquireData(input) {
 const lines = input.split("\n");
 return lines
    .slice (1)
    .filter (line => line.trim() !== "")
    .map   (line => line.split(","))
    .filter (fields => fields[1].trim() === "India")
    .map   (fields => ({city: fields[0].trim(), phone: fields[2].trim()}))
    ;
}
```

我还想过是否要内联 lines 变量，但我感觉它还算能解释该行代码的意图，因此我还是将它留在了原处。

延伸阅读

如果想了解更多用集合管道替代循环的案例，可以参考我的文章“Refactoring with Loops and Collection Pipelines”[mf-ref-pipe]。

## 8.9 移除死代码（Remove Dead Code）

```js
if (false) {
  doSomethingThatUsedToMatter();
}
```

### 动机

事实上，我们部署到生产环境甚至是用户设备上的代码，从来未因代码量太大而产生额外费用。就算有几行用不上的代码，似乎也不会因此拖慢系统速度，或者占用过多的内存，大多数现代的编译器还会自动将无用的代码移除。但当你尝试阅读代码、理解软件的运作原理时，无用代码确实会带来很多额外的思维负担。它们周围没有任何警示或标记能告诉程序员，让他们能够放心忽略这段函数，因为已经没有任何地方使用它了。当程序员花费了许多时间，尝试理解它的工作原理时，却发现无论怎么修改这段代码都无法得到期望的输出。

一旦代码不再被使用，我们就该立马删除它。有可能以后又会需要这段代码，但我从不担心这种情况；就算真的发生，我也可以从版本控制系统里再次将它翻找出来。如果我真的觉得日后它极有可能再度启用，那还是要删掉它，只不过可以在代码里留一段注释，提一下这段代码的存在，以及它被移除的那个提交版本号——但老实讲，我已经记不得我上次撰写这样的注释是什么时候了，当然也未曾因为不写而感到过遗憾。

在以前，业界对于死代码的处理态度多是注释掉它。在版本控制系统还未普及、用起来又不太方便的年代，这样做有其道理；但现在版本控制系统已经相当普及。如今哪怕是一个极小的代码库我都会把它放进版本控制，这些无用代码理应可以放心清理了。

### 做法

如果死代码可以从外部直接引用，比如它是一个独立的函数时，先查找一下还有无调用点。

将死代码移除。

测试。
