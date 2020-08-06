# typescript基础

## 枚举

**定义**

```typescript
enum Color { Red, Green, Blue }
let c: Color = Color.Green;
```

默认情况下，枚举从0开始对其成员编号。您可以通过手动设置其成员之一的值来更改此值。 例如，我们可以从1开始而不是从0开始.

```typescript
enum Color { Red = 1, Green, Blue }
let c: Color = Color.Green;
```

或者，甚至手动设置枚举中的所有值.

```typescript
enum Color { Red = 1, Green = 2, Blue = 4 }
let c: Color = Color.Green;
```

枚举的一个方便功能是，您也可以在枚举中从数字值转到该值的名称。 例如，如果我们具有值2，但不确定上面的Color枚举中映射到的值，我们可以查找对应的名称:

```typescript
enum Color { Red = 1, Green, Blue }
let colorName: string = Color[2];
console.log(colorName); // Displays 'Green' as its value is 2 above
```

## Any

来自动态内容，例如 来自用户或第三方库。 在这些情况下，我们要选择退出类型检查，并让值通过编译时检查。 为此，我们将它们标记为任何类型:

```typescript
let notSure: any = 4;
notSure = "maybe a string instead";
notSure = false; // okay, definitely a boolean
```

any类型是使用现有JavaScript的强大方法，可让您在编译过程中逐步选择加入和选择退出类型检查。 您可能希望Object像其他语言一样扮演相似的角色。 但是，对象类型的变量仅允许您为其分配任何值。 您不能在它们上调用任意方法，即使是实际存在的方法.

```typescript
let notSure: any = 4;
notSure.ifItExists(); // okay, ifItExists might exist at runtime
notSure.toFixed(); // okay, toFixed exists (but the compiler doesn't check)

let prettySure: Object = 4;
prettySure.toFixed(); // Error: Property 'toFixed' doesn't exist on type 'Object'.
```

> **注意：如我们的‘’允许和不允许“部分所述，避免使用支持非原始对象类型的对象。**

如果您知道类型的某些部分，但可能不是全部，那么any类型也很方便。 例如，您可能有一个数组，但该数组混合了不同类型：

```typescript
let list: any[] = [1, true, "free"];
list[1] = 100;
```

## Void

void有点像任何其他的相反：根本没有任何类型。 您可能通常将其视为不返回值的函数的返回类型:

```typescript
function warnUser(): void {
  console.log("This is my warning message.")
}
```

声明类型为void的变量没有用，因为您只能将null赋值（仅当未指定--strictNullChecks时，请参阅下一节）或未定义:

```typescript
let unusable: void = undefined;
unusable = null; // OK if `--strictNullChecks` is not given
```

## Null and Undefined

在TypeScript中，undefined和null实际上实际上都有自己的类型，分别命名为undefined和null。 就像`void`一样，它们本身并不是非常有用:

```typescript
// Not much else we can assign to these variables!
let u: undefined = undefined;
let n: null = null;
```

默认情况下，null和undefined是所有其他类型的子类型。 这意味着您可以将null和undefined分配给数字.

但是，当使用--strictNullChecks标志时，null和undefined仅可分配给任何类型及其各自的类型（一个例外是undefined也可分配给void）。 这有助于避免许多常见错误。 如果要传递字符串或null或undefined，则可以使用联合类型string | null | undefined.

## Never

`never`类型表示永不出现的值的类型。 例如，永远不会抛出异常的函数表达式或箭头函数表达式的返回类型，或者永远不会返回异常的箭头函数表达式； 变量也永远不会被任何永远不可能为真的类型防护所缩小.

`never`类型是每种类型的子类型，并且可以赋值给每种类型。 但是，任何类型都不是（永远不会除外的）永不的子类型或可赋值给它的子类型。 甚至任何东西都无法赋值给`never`。

一些函数返回值是`never`的示例：

```typescript
function error(message: string): never {
  throw new Error(message);
}

function fail() {
  return error("Somethine failed.")
}

function infiniteLoop(): never {
  while (true) {
    
  }
}
```

## Object

```typescript
declare function create(o: object | null): void;

create({ prop: 0 }); // OK
create(null); // OK

create(42); // Error
create("string"); // Error
create(false); // Error
create(undefined); // Error
```

## typescript断言

有时，您会遇到比TypeScript更了解值的情况。 通常，当您知道某个实体的类型可能比其当前类型更具体时，就会发生这种情况。

类型断言是一种告诉编译器“相信我，我知道我在做什么”的方法。 类型断言就像其他语言中的类型转换一样，但是不执行数据的特殊检查或重组。 它对运行时间没有影响，仅由编译器使用。 TypeScript假设您（程序员）已经执行了所需的任何特殊检查。

类型断言有两种形式。 一种是“尖括号”语法：

```typescript
let someValue: any = "this is a string";
let strLength: number = (<string>someValue).length;
```

也可以使用`as`语法：

```typescript
let someValue: any = "this is a string";
let strLength: number = (someValue as string).length;
```

