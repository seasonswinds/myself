# Netty

### ByteBuffer有哪些缺点？

- 长度固定，不能动态扩容、缩容
- 只有一个标识位置的指针position，读写的时候需要手动调用flip()、rewind()等，很容易导致程序处理失败
- API功能有限

