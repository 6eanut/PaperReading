# Implementation

communication layer是c实现，其余是rust实现

实现了Ref `<T>`, MutRef `<T>`, DBox`<T>`, 分别对应&T， &mut T， Box`<T>`

针对上述三个type，还实现了copy，clone，drop

还更改了编译器，适配drust，比如reference和box的位数还有color

C方面实现一堆看不懂的东西，主要就是说了control plane和data plane

heap allocator piggyback

scheduler tokio

# Limitations

对于unsafe code，drust没法保证一致性，这个需要开发人员考虑

如果说有太多mutex，即太多共享数据，那性能也不太好

不支持ASLR，但是以后可以支持
