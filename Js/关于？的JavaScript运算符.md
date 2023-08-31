# 关于 ？的 Javascript 运算符

> 文中的nullish 代表 null 或者 undefined

## 1. ?? 操作符

在 JavaScript 中，`??` 操作符被称为`nullish` 合并操作符。如果第一个参数不是 `null/undefined`，这个运算符将返回第一个参数，否则，它将返回第二个参数。让我们看一个例子。

```js
null ?? 5 // => 5
3 ?? 5 // => 3
```

当为一个变量分配默认值时，JavaScript 开发人员传统上依赖于逻辑 `OR` 操作符，如下所示。

```js
var prevMoney = 1
var currMoney = 0
var noAccount = null
var futureMoney = -1

function moneyAmount(money) {
  return money || `You currently do not own an account in the bank`
}

console.log(moneyAmount(prevMoney)) // => 1
console.log(moneyAmount(currMoney)) // => `You currently do not own an account in the bank`
console.log(moneyAmount(noAccount)) // => `You currently do not own an account in the bank`
console.log(moneyAmount(futureMoney))//  => -1
```

上面我们创建了一个函数 `moneyAmount`，负责返回用户的当前余额。**我们使用了 `||` 操作符来识别没有帐户的用户。当 money 为 0 或者 null 的时候都会返回在当前银行没有这个账户，但是实际上账户是可能为 0 的 。在上面的示例中， `||` 操作符将 0 视为假值，因此识别不出来我们的用户拥有一个 0 美元的帐户。** 让我们通过使用 nullish 合并操作符来解决这个问题。

```js
var currMoney = 0
var noAccount = null

function moneyAmount(money) {
  return money ?? `You currently do not own an account in the bank`
}
 moneyAmount(currMoney) // => 0
 moneyAmount(noAccount) // => `You currently do not own an account in the bank`
```

**总结： `??` 操作符允许我们分配默认值，同时忽略像 0 和空字符串这样的假值。**

## 2. `??=` 操作符

`??=` 又称为逻辑 nullish 赋值操作符，与我们之前学到的内容密切相关。让我们看看它们是如何联系在一起的。

```js
var x = null
var y = 5

console.log(x ??= y) // => 5
console.log(x = (x ?? y)) // => 5
```

只有当前值为 null 或 undefined 时，此赋值运算符才会分配新值。

上面的例子强调了这个操作符如何实质上是 nullish 赋值的语法糖。

接下来，让我们看看这个操作符与默认参数的区别。

```js
function gameSettingsWithNullish(options) {
  options.gameSpeed ??= 1
  options.gameDiff ??= 'easy'
  return options
}

function gameSettingsWithDefaultParams(gameSpeed=1, gameDiff='easy') {
  return {gameSpeed, gameDiff}
}

gameSettingsWithNullish({gameSpeed: null, gameDiff: null}) // => { gameSpeed: 1, gameDiff: 'easy' }
gameSettingsWithDefaultParams(null, null) // => { gameSpeed: null, gameDiff: null }
```

上面的函数处理空值的方式有一个显著的不同。默认参数将使用 null 参数覆盖默认值，nullish 赋值操作符不会。默认参数和 nullish 赋值都不会覆盖未定义的值。

## 3. `?.` 操作符

可选的链接操作符 `?.` 允许开发人员读取深度嵌套在一个对象链中的属性值，而不必沿途显式验证每个引用。当引用为 null 时，表达式停止计算并返回 `undefined`，让我们来看一个例子。

```js
var travelPlans  = {
  destination: 'DC',
  monday: {
    location: 'National Mall',
    budget: 200
  }
};

const tuesdayPlans = travelPlans.tuesday?.location;
console.log(tuesdayPlans) // => undefined
```

现在，让我们把迄今为止所学到的一切结合起来，把星期二添加到我们的新旅行计划中去吧！

```js
function addPlansWhenUndefined(plans, location, budget) {
  if (plans.tuesday?.location === undefined) {
    var newPlans = {
      plans,
      tuesday: { location: location ?? "Park", budget: budget ?? 200 },
    };
  } else {
    newPlans ??= plans; //will only override if newPlans is undefined
    console.log("Plans have already been added!");
  }
  return newPlans;
}

var newPlans = addPlansWhenUndefined(travelPlans, "Ford Theatre", null);
console.log(newPlans) // => { plans:
                  //{ destination: 'DC',
                  // monday: { location: 'National Mall', budget: 200 } },
                  // tuesday: { location: 'Ford Theatre', budget: 200 } }

newPlans = addPlansWhenUndefined(newPlans, null, null) // logs => Plans have already been added!
                                                      // returns => newPlans object
```

我们现在已经创建了一个函数，该函数将计划添加到当前没有嵌套属性 `tuesday.location` 的对象中。我们还使用 nullish 操作符提供默认值。这个函数将接受错误的值，如“0”作为有效的参数。这意味着我们的预算可以设置为零，没有任何错误。

## 4. `?` 操作符

三元运算符 `?` 有三个操作数: 一个条件，一个条件为真时执行的表达式，以及一个条件为假时执行的表达式。让我们看看它是如何运作的。

```js
function checkCharge(charge) {
return (charge > 0) ? 'Ready for use' : 'Needs to charge'
}

console.log(checkCharge(20)) // => 'Ready for use'
console.log(checkCharge(0)) // => 'Needs to charge'
```

如果你花了一些时间研究 JavaScript，你可能以前见过三元运算符。然而，你知道三元运算符可以用于变量赋值吗？

```js
var budget = 0
var transportion = (budget > 0) ? 'Train' : 'Walking'
console.log(transportion) // => 'Walking'
```

我们甚至可以用它来复制 nullish 赋值的行为。

```js
var x = 6
var x = (x !== null || x !== undefined) ? x : 3
console.log(x) // => 6
```

现在让我们通过创建一个函数来泛化这种行为！

```js
function nullishAssignment(x,y) {
  return (x == null || x == undefined) ? y : x
}

var x = nullishAssignment(null, 8) // => 8
var y = nullishAssignment(4,8) // => 4
```

在结束之前，让我们使用三元运算符来重构前面示例中的函数。

```js
function addPlansWhenUndefined(plans, location, budget) {
  var newPlans =
    plans.tuesday?.location === undefined
      ? {
          ...plans,
          tuesday: { location: location ?? "Park", budget: budget ?? 200 },
        }
      : console.log("Plans have already been added!");
  newPlans ??= plans;
  return newPlans;
}
```

