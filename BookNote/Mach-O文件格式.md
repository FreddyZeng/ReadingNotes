# Mach-O文件格式

https://github.com/aidansteele/osx-abi-macho-file-format-reference

https://adrummond.net/posts/macho

https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html#//apple_ref/doc/uid/20001860-BAJGJEJC



Mach-O 是一个可执行文件格式，是一个很好的传输代码格式。一个可执行文件格式决定了把二进制中的代码和数据读取进内存的顺序。代码和数据的顺序影响了内存的占用和内存分页，因此直接影响了程序的性能。

Mch-O库是由段组成的，每一个segment都包含一个或者多个section。代码和数据放在不同的section。segment 总是从page boundary（页边界）开始，但是section不必要page-aligned（页对齐）。segment的size是通过所有的section字节数来确定的。segment的大小是按照4096 字节倍数来进行组成大小的，最小值是4096字节。

segment 和section的命名是根据使用意图命名的。segment命名是双







