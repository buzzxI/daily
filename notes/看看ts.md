# hello world

```typescript
const msg: string = 'hello world';
consolg.log(msg);
```

>   相比于 js 添加了 string 类型进行约束

# 基本数据类型

*   any: 表示任意类型

*   number: 同时可以表示整数和小数

    ```typescript
    let num: number = 10;
    let points: number = 1.1;
    ```

*   string: 字符串类型，可以使用单引号或双引号，使用反引号 \`` 可以用来拼接字符串

    ```typescript
    let num: number = 10;
    let s: string = `the num is ${ num }`;
    ```

    这种引号的拼接方式不是必须的，但却是 eslint 中建议的

    对于不拼接的类型 eslint 建议使用单引号

*   boolean: 布尔类型

*   []: 数组类型

    ```typescript
    // 只是在基础数据类型后面加上[]
    let arr: number[] = [1,2];
    // 使用数组泛型类型
    let nums: Array<number> = [1,2];
    ```

    具体的可以看后面的[数组](#数组)

*   

# splice 函数

主要是因为在 js/ts 中操作数组实在是太随意了，随便的进行添加和删除，默认的话就可以实现栈的功能(pop 和 poll)

这个 splice 函数准确来说是用来修改数组中的某些项的

```typescript
array.splice(index, howMany, [element1][, ..., elementN]);
```

表示在数组的第 index 下标开始，向后删除 howMany 个元素(包括 index 位置的元素)，并从这个位置开始添加 element1，element2...elementN

比如：

```typescript
const arr = [1,2,3,4,5,6];
arr.splice(0, 2, 7, 8, 9);
arr.forEach((num) => {
    console.log(num);
})
```

打印结果：

```shell
7
8
9
3
4
5
6
```

# 模块

## 导入

通过 export 关键字，在 ts 中可以导出: 类、接口、方法、变量

比如：

```typescript
// 导入接口
export interface StringValidator {
    isAcceptable(s: string): boolean;
}
// 导出变量
export const numberRegexp = /^[0-9]+$/;
// 导出类
export class ZipCodeValidator implements StringValidator {
    isAcceptable(s: string) {
        return s.length === 5 && numberRegexp.test(s);
    }
}
// 导出方法
export function validate(param: string): boolean{
    if (param.length === 0 || param) return false;
    return true;
};
```

而习惯性的，都是在文件结尾进行导出，比如:

```typescript
function validate(param: string): boolean{
    if (param.length === 0 || param) return false;
    return true;
};

export { validate }
```

更进一步的，在可以在导出时进行重命名:

```typescript
function validate(param: string): boolean{
    if (param.length === 0 || param) return false;
    return true;
};
// 将方法以 validator 的名字导出
export { validate as validator };
```

## 导入

类似的使用 import 关键字进行导入，也可以进行重命名

```typescript
// 再把导出名 validator 重命名为 validate
import { validator as validate } from './Validator'

const name: string = 'abc';

console.log(validate(name));
```

## 默认导出 

export default 关键字，**此时不要使用大括号**

>   导入的时候也不要使用大括号

每个模块仅有一个默认导出

```typescript
// Validator.ts
function validate(param: string): boolean{
    if (param.length === 0 || param) return false;
    return true;
};

export default validate;

// test.ts
// 注意导入的时候不需要大括号，如果非要使用的话，需要下面的那种写法
import  validate from "./Validator";
//import { default as validate} from "./Validator";

const name: string = 'abc';

console.log(validate(name));
```

