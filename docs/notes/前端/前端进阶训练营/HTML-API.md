# HTML 其他API

## Range Api

`Range` 接口表示一个包含节点与文本节点的一部分的文档片段。

可以用 `Document` 对象的 `Document.createRange` 方法创建 `Range`，也可以用 `Selection` 对象的 `getRangeAt` 方法获取 `Range`。另外，还可以通过 `Document` 对象的构造函数 `Range()` 来得到 `Range`。