## Fifo

最简单的 ipc kernel object，功能和结构都十分普通，完全符合模板。

特点：

* 创建时指定了每个 element 的大小和 fifo 最大长度。
  * 读写以 element 为基本单位
  * 最大长度有限，超出则写入失败。
* first in first out 不是特点，其他 ipc 模块也是 fifo 的。
* 内存大小，不需要动态变化，高效。