# arceos-stage4-blog
## 序言
首先，非常高兴能够进入开源操作系统训练营第四阶段的学习，和诸多优秀的同学交流技术，让我对操作系统诸多底层细节实现有了更深刻的理解，在动手实践的过程中，也和老师们交流了许多，感谢陈渝老师在周会中给予的指引，也感谢郑友捷老师在宏内核项目上的许多解答，也感谢郑植陈宏同学在合并下游仓库过程中的指点交流，诸多种种令我受益匪浅。

## 具体工作
本次阶段四总共四周，以下按照时间线陈述我的工作内容

### 第一周
由于我完成二三阶段的时间比较晚了，几乎是踩着ddl加入的第四阶段，加上在和陈老师的讨论中我表达了自己想更熟悉操作系统底层的一些具体实现，第一周陈老师给我安排的任务是给allocator模块编写说明文档，在这个过程中一方面锻炼我对代码工程的阅读能力，也一方面希望这样一份工作能为后来者提供便利，更好地了解arceos这样一个组件化操作系统。

在第一周主要还是以阅读代码为主，并基于deepwiki给出的指引写了一份简短的项目结构简介

### 第二周
在第一周的周会结束后，和老师交流之后我就开始思考，以一个刚入门的初学者，怎样的文档才能让我更直观地学习一个项目模块呢？我选择从两方面入手，一方面以开发者的视角直观地展示allocator这个模块的功能实现过程;另一方面我以一个使用者的视角给出一个具体使用的代码展示，以测试的方式展示算法的特性，下面以其中的buddy分配算法为例简单介绍：

由于算法本身特性：将内存划分为大小为 2^n 的块，分配时，系统查找最小能满足请求的块；如果块过大，则递归分裂为两个大小相等的“伙伴”块。释放时，系统检查相邻伙伴块是否空闲，若空闲则合并为更大的块，由此在编写文档和测例时，着重展现大块分裂与合并的过程：

以下是一个简单的使用示例以及结果输出
```rust
// test3.rs
extern crate alloc;

use alloc::boxed::Box;
use allocator::BuddyByteAllocator;
use core::alloc::Layout;
use allocator::ByteAllocator;
use allocator::BaseAllocator;

// 创建堆内存池（避免栈溢出）
fn create_test_pool(size: usize) -> Box<[u8]> {
    vec![0u8; size].into_boxed_slice()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fragmentation_handling() {
        const HEAP_SIZE: usize = 4 * 1024 * 1024; // 4MB
        let heap_mem = create_test_pool(HEAP_SIZE);
        let heap_start = heap_mem.as_ptr() as usize;
        let mut allocator = BuddyByteAllocator::new();
        
        unsafe {
            allocator.init(heap_start, heap_mem.len());
        }

        // 分配多个小块（制造碎片）
        let mut ptrs = Vec::new();
        let small_layout = Layout::from_size_align(4096, 4096).unwrap(); // 4KB
        
        for _ in 0..100 {
            let ptr = allocator.alloc(small_layout).expect("小块分配失败");
            ptrs.push(ptr);
        }
        
        // 释放所有奇数索引的块
        for i in (1..ptrs.len()).step_by(2) {
            allocator.dealloc(ptrs[i], small_layout);
        }
        
        // 尝试分配大块（应能利用碎片合并）
        let large_size = 1024 * 1024; // 1MB
        let large_layout = Layout::from_size_align(large_size, large_size).unwrap();
        assert!(
            allocator.alloc(large_layout).is_ok(),
            "应能利用碎片分配{}字节大块",
            large_size
        );
        
        // 清理剩余内存
        for i in (0..ptrs.len()).step_by(2) {
            allocator.dealloc(ptrs[i], small_layout);
        }
    }
}
```

类似地，针对bitmap、slab、tlsf算法也编写了对应算法特性的测例