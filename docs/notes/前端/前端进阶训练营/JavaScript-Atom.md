# JavaScript Atom

## Unicode字符集

详细介绍网站：[https://www.fileformat.info/info/unicode/](https://www.fileformat.info/info/unicode/)

> ⚠️:Unicode是字符集，Utf-8、Utf-16是编码方式。

**以Blocks纬度分类：**

[https://www.fileformat.info/info/unicode/block/index.htm](https://www.fileformat.info/info/unicode/block/index.htm)

- 0 ~ U+007F：常用拉丁字符（兼容ASCII编码）
  
  `String.fromCharCode(num)`

- U+4E00 ~ U+9FFF：CJK `C`hinese`J`apanese`K`orean三合一
  
  有一些增补区域（extension）

- U+0000 - U+FFFF：[BMP](https://zh.wikipedia.org/wiki/Unicode字符平面映射) 基本平面

**以Categories纬度分类：**

[https://www.fileformat.info/info/unicode/category/index.htm](https://www.fileformat.info/info/unicode/category/index.htm)

[space空格系列](https://www.fileformat.info/info/unicode/category/Zs/list.htm)

## Atom

### whiteSpace

可查阅 unicode [space列表](https://www.fileformat.info/info/unicode/category/Zs/list.htm)

- Tab：制表符（打字机时代：制表时隔开数字很方便）
- VT：纵向制表符
- FF: FormFeed
- SP: Space
- NBSP: NO-BREAK SPACE（和 SP 的区别在于不会断开、不会合并）
- ...

### LineTerminator 换行符

- LF: Line Feed `\n`
- CR: Carriage Return `\r`
- ...

### Comment 注释

### Token 记号：一切有效的东西

- Punctuator: 符号 比如 `> = < }`

- Keywords：比如 `await`、`break`... 不能用作变量名，但像 getter 里的 `get`就是个例外

  - Future reserved Keywords: `eumn`

- IdentifierName：标识符，可以以字母、_ 或者 $ 开头，代码中用来标识**[变量](https://developer.mozilla.org/en-US/docs/Glossary/variable)、[函数](https://developer.mozilla.org/en-US/docs/Glossary/function)、或[属性](https://developer.mozilla.org/en-US/docs/Glossary/property)**的字符序列

  - 变量名：不能用 Keywords
  
  - 属性：可以用 Keywords
  
- Literal: 直接量

  - Number
    - 存储 Uint8Array、Float64Array
    - 各种进制的写法
      - 二进制0b
      - 八进制0o
      - 十进制0x
    - 实践
      - 比较浮点是否相等：Math.abs(0.1 + 0.2 - 0.3) <= Number.EPSILON
      - 如何快捷查看一个数字的二进制：(97).toString(2)
  - String
    - Character
    - Code Point
    - Encoding
      - unicode编码 - utf
        - utf-8 可变长度 （控制位的用处）
    - Grammar
      - `''`、`""`、``` `
  - Boolean
  - Null
  - Undefind