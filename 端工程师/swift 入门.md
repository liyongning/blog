写给前端工程师的 Swift 入门教程

通过 TS 和 Swift 类比的方式快速入门 Swift。

以下内容都需要亲自在 Xcode 中敲一遍，以提升熟练度和加深印象（看三遍不如写一遍），一定要亲自练。

打开 Xcode => Create New Project => macOS => Command Line Tool => 填充项目名称和组织信息（随便填）=> Next 选择好存放目录 => 创建成功。接下来在 main 文件中编写代码，保存后点击侧边栏上的三角标即可构建运行，也可以使用 Command + R 快捷键，你就能在 terminal 中看到对应的内容了。

按照传统，一门新语言的第一个程序应该在屏幕上打印出 "Hello World" 这个字符串

Swift 语法：

```swift
print("Hello World")
```

Typescript 语法：

```typescript
console.log("Hello World")
```

## 简单值

Swift 使用 let 来创建常量，var 创建变量，可以类比 TS 中的 const 和 let；另外 Swift 和 TS 一样也存在潜在的类型推断，即根据值推断变量的类型，同样的变量类型一旦确定就不可再变。

```swift
// 声常量，声明之后就不能变了
let myConstant = 42
// 下面这行编译器会报错
// myConstant = 50

// 声明变量，之后可以重新赋值
var myVariable = 42
myVariable = 50

var myVariableStr: String
myVariableStr = "test"
// 下面这行编译器会报错
// myVariableStr = 10
```

```typescript
const myConstant = 42
let myVariable = 42
myVariable = 50
```

Swift 中的值永远不会被做隐式类型转换，这点和 Typescript 不一样，如果需要将值转换成其它类型，需要显示创建目标类型的实例。

```swift
let label = "The width is "
let width = 94
let widthLabel = label + String(width)
print("widthLabel = ", widthLabel)
// The width is 94

// 下面这行编译器会报错
// let widthLabelCompilerErr = label + width
```

```typescript
const label = "The width is "
const width = 94
const widthLabel = label + String(width)
console.log("widthLabel = ", widthLabel)
// The width is 94

// 下面这样是正确的，会走隐式类型转换 => number 转成 string
const widthLabelTest = label + width
console.log("widthLabelTest = ", widthLabelTest)
```

Swift 中也可以像 TS 的模板字符串一样在字符串中嵌入变量（值）：将值写在括号内，并在括号前加上反斜杠（\）。

```swift
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
print("I have \(apples) apples.")
```

```typescript
const apples = 3
const oranges = 5
const appleSummary = `I have ${apples} apples.`
const fruitSummary = `I have ${apples + oranges} pieces of fruit.`
console.log(`I have ${apples} apples.`)
```

对于占多行的字符串，直接使用三个双引号（"""）。只要每行开头的索引与闭合引号的索引一致，该行的缩进就会被移除，这点 TS 是没有的能力，TS 要实现类似的能力得通过运行动态生成字符串的形式才行。

```swift
let apples = 3
let oranges = 5
let quotation = """
  Even though there's whitespace to the left,
  the actual lines aren't indented.
      Except for this line.
  Double quotes (") can appear without being escaped

  I still have \(apples + oranges) pieces of fruit
  """
print(quotation)
```

```typescript
const apples = 3
const oranges = 5
const quotation = `
  Even though there's whitespace to the left,
  the actual lines aren't indented.
      Except for this line.
  Double quotes (") can appear without being escaped

  I still have \(apples + oranges) pieces of fruit
  `

// 输出的内容不是定格显示的
console.log(quotation)

// 需要通过下面的方式才可以达到和 swift 一样的效果
const strArr = [
    "Even though there's whitespace to the left,",
    "the actual lines aren't indented.",
    "    Except for this line.",
    'Double quotes (") can appear without being escaped',
    "",
    `I still have ${apples + oranges} pieces of fruit`
]
const quotationStr = strArr.join("\n")
console.log(quotationStr)
```

接下来是 Swift 中的数组和字典，可以类比 TS 中的数组和对象。

```swift
// 自动推导为 [String]，即 var fruits: [String] = []
var fruits = ["strawberries", "limes", "tangerines"]
fruits[1] = "grapes"
// 添加新元素时，数组会自动扩容
fruits.append("blueberries")

// 这里只能显示声明为 [Any] 才行，这点使用上和 TS 有点区别
var testArr: [Any] = [1, "test"]

// 空数组
var number: [Int] = []

// 常量数组，这点和 typescript 也不一样
let arr = [1]
// 下面这行编译器会报错
// arr.append(2)
```

类比 Typescript 数组

```typescript
// 自动推导为 string[]，即 var fruits: string[] = []
let fruits = ["strawberries", "limes", "tangerines"]
fruits[1] = "grapes"
fruits.push("apple")
// typescript 中也可以，但 swift 不行，运行时会报索引超出范围的错误
fruits[4] = "orange"

// 混合类型数组，会自动推导为 const testArr: Array<number | string>
const testArr = [1, "test"]

// 空数组
let numArr: number[] = []


// const 声明的数组，依旧可以更改数组的大小和其中的元素，这点和 swift 不一样
testArr[2] = 2
```

对象

```swift
var occupations = [
    "key": "value",
    "Malcolm": "Captain"
]
// 同样的，如果是 const 声明的对象，下面这行一样编译报错
occupations["Jayne"] = "Public Relations"

// 声明空对象
var emptyObj: [String: Double] = [:]
```

```typescript
const occupations = {
    "key": "value",
    "Malcolm": "Captain"
}
// 即使 const 声明的对象，也可以更改其中的属性和值，和 swift 不一样
occupations["Jayne"] = "Public Relations"

// 声明空对象
let emptyObj = {}
```

## 控制流

使用 if 和 switch 来构建条件语句，使用 for-in、while i级 repeat-while 来构建循环。邮件或循环变量周围的括号可以省略，但代码块主体的大括号必须存在，比如：

```swift
let individualScores = [75, 43, 103, 87, 12]
for score in individualScores {
    // if 语句的条件必须是一个 boolean 值，和 TS 不一样，不存在隐式类型转换
    if score > 100 {
        print(score)
    }
}
```

### if 表达式

可以在表达式等号（=）后面跟 if 或 switch 语句，根据条件选择不同的值

```swift
let teamScore = 11
let scoreDecoration = if teamScore > 10 {
    "🎉"
} else {
    ""
}
// 🎉
print(scoreDecoration)
```

TS 中没有对应的原生能力，但三元表达式比较接近，以及可以通过立即执行函数来模拟类似的效果。

```typescript
const teamScore = 11
const scoreDecoration = teamScore > 10 ? "🎉" : ""
```

```typescript
const teamScore = 11
const scoreDecoration = (() => {
    if (teamScore > 10) return "🎉"
    else return ""
})()
```

### 可选值和解包

可以把 if 和 let 结合起来处理可选类型的值

```swift
// 在类型的后面加上问号（？）即可将该变量的值标记为可选值，不存在时为 nil
var optionalName: String? = "John"
var greeting = "Hello"
// 这行承担了三个职责：
//  1. 条件判断：可选值 optionalName 不为 nil，则 if 条件成立，一是可以进入当前代码块，而是可以进入下面两步
//  2. 解包：optionalName 是一个可选值，只有确定值确实存在，才会执行相应的操作，比如赋值、if 条件成立
//  3. 声明新变量：将 optionalName 解包后的具体值赋值给声明的 name 变量；name作用于只限于 if 表达式内，类似 TS 块级作用域内的变量
if let name = optionalName {
  greeting = "Hello \(name)"
}

print(greeting)
```

可选值还有其他使用方式，比如：

```swift
// 更简短的使用方式，该解包后的值使用相同的变量名称
var optionalName: String? = "John"
// 注意这里不能省略 let，省略 let 就会违背 swift if 条件必须是布尔表达式的原则
if let optionalName {
    print("Hey, \(optionalName)")
}

// 使用 ?? 运算符提供默认值，如果可选值缺失，则该用默认值，TS 中也有类似的语法
let fullName = "John Appleseed"
print("Hi \(optionalName ?? fullName)")

// 强制断言
var obj: [String: String] = [:]
obj["test"] = "test value"
// 没有断言，编译器会提示 warning，不能把 String? 赋值给 Any
print(obj["test"]!)
```

### switch 语句

Switch 支持任意类型的数据以及多种比较操作，并非仅限于整数和相等性测试。

```swift
let vegetable = "red pepper"
switch vegetable {
// 单值的相等匹配
case "celery":
    print("Add some raisins and make ants on a log.")
    // 不需要手动 break，匹配成功自动停止不会穿透，这点和 TS 不一样；
    // 如果要穿透需要手动添加 fallthrough 关键字
// 多值匹配
case "cucumber", "watercress":
    print("That would make a good tea sandwich.")
// 如果条件匹配会将 vegetable 的绑定到变量 x 上
case let x where x.hasSuffix("pepper"):
    print("Is it a spicy \(x)?")
// case 其它条件写法可自行探索
// 和 TS 一样，需要默认穷尽所有情况，所以需要 default 兜底
default:
    print("Everything tastes good in soup.")
}
```

### 循环

可以使用 for-in 来便利字典和数组中的项，类似 TS 中的 for-of 用法。

```swift
// 字典是无序的，所以键值遍历顺序是任意的
let interestingNumbers = [
    "Prime": [2, 3, 5, 7, 11, 13],
    "Fibonacci": [1, 1, 2, 3, 5, 8],
    "Square": [1, 4, 9, 16, 25],
]
var largest = 0
for (key, numbers) in interestingNumbers {
    print("key: \(key)")
    print("value: \(numbers)")
    
    for number in numbers {
        if number > largest {
            largest = number
        }
    }
}
print(largest)
```

可以使用`..<`生成一系列索引来在循环中保留索引：

```swift
var total = 0
// ..< 相当于 [0, 1, 2, 3]，... 相当于 [0, 1, 2, 3, 4]
for i in 0..<4 {
    print("i = \(i)")
}
```

 使用 while 或者 repeat while 来重复执行一段代码，直到条件不匹配后结束循环。swift while 和 TS 中的 while 一样，repeat while 和 TS 中的 do while 一样，至少先执行一次循环。

```swift
var n = 2
while n < 100 {
    n *= 2
}
// 128
print(n)

var m = 2
repeat {
    m *= 2
} while m < 100
// 128
print(m)
```

## 函数和闭包
