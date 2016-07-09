---
title: Go性能优化技巧
date: 2016-06-02 18:12:09
tags: Go
---
## 字符串(string)和字节数组（slice, [ ]byte）转换时需付出 “沉重” 代价。
在对性能有要求或者操作频繁的地方使用如下形式:
```Go
import (
	"strings"
	"unsafe"
)
func str2bytes(s string) []byte {
	x:= (*[2]uintprt)(unsafe.Pointer(&s))
	h:= [3]uintprt{x[0],x[1],x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}
func bytes2str(b[] byte) string{
	return *(*string)(unsafe.Pointer(&b))
}
```
参考：http://studygolang.com/topics/1615

## array vs slice
Go array 以 pass-by-value 方式传递, 整个 array 函数完全在栈上完成，而 slice 函数则需执行 makeslice，继而在堆上分配内存, 对于大的对象，建议使用slice 代替 array，而短小的对象，复制成本远小于在堆上分配和回收操作，因此建议用array。

参考: http://studygolang.com/topics/1679
## map优化
1. 预设足够的容量空间
map 会按需扩张，但须付出数据拷贝和重新哈希成本。如有可能，应尽可能预设足够容量空间，避免此类行为发生。
2. 直接存储
对于小对象，直接将数据交由 map 保存，远比用指针高效。这不但减少了堆内存分配，关键还在于垃圾回收器不会扫描非指针类型 key/value 对象。
3. 空间收缩
map 不会收缩 “不再使用” 的空间。就算把所有键值删除，它依然保留内存空间以待后用。可以：
* mapVar = nil
* 使用新的map替换原来的
4. int key 要比 string key 更快
参考: http://studygolang.com/topics/1682

## 滥用defer，可能会造成资源泄露，需注意！
参考：http://studygolang.com/topics/1685

## 闭包，会变成额外成本在运行期由 CPU、runtime 负担。甚至因不合理使用，造成性能问题
参考：http://studygolang.com/topics/1690

## 总结： 个人认为这些性能的问题，大部分情况下不需要做过份的追求，只是在某些特殊的场景下，比如频繁调用，对性能有很高的要求时，可以参考一二！！！
