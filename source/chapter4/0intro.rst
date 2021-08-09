引言
==============================

本章导读
-------------------------------

..
  chyyuu：有一个ascii图，画出我们做的OS。


本章展现了操作系统一系列功能：

- 通过动态内存分配，提高了应用程序对内存的动态使用效率
- 通过页表的虚实内存映射机制，简化了编译器对应用的地址空间设置
- 通过页表的虚实内存映射机制，加强了应用之间，应用与内核之间的内存隔离，增强了系统安全
- 通过页表的虚实内存映射机制，可以实现空分复用（提出，但没有实现）

.. _term-illusion:
.. _term-time-division-multiplexing:
.. _term-transparent:

上一章，我们分别实现了多道程序和分时多任务系统，它们的核心机制都是任务切换。由于多道程序和分时多任务系统的设计初衷不同，它们在任务切换的时机和策略也不同。有趣的一点是，任务切换机制对于应用是完全 **透明** (Transparent) 的，应用可以不对内核实现该机制的策略做任何假定（除非要进行某些针对性优化），甚至可以完全不知道这机制的存在。

在大多数应用（也就是应用开发者）的视角中，它们会独占一整个 CPU 和特定（连续或不连续）的内存空间。当然，通过上一章的学习，我们知道在现代操作系统中，出于公平性的考虑，我们极少会让独占CPU这种情况发生。所以应用自认为的独占CPU只是内核想让应用看到的一种 **幻象** (Illusion) ，而 CPU 计算资源被 **时分复用** (TDM, Time-Division Multiplexing) 的实质被内核通过恰当的抽象隐藏了起来，对应用不可见。

与之相对，我们目前还没有对内存管理功能进行有效的管理，仅仅是把程序放到某处的物理内存中。在内存访问方面，所有的应用都直接通过物理地址访问物理内存，这使得应用开发者需要了解繁琐的物理地址空间布局，访问内存也很不方便。在上一章中，出于任务切换的需要，所有的应用都在初始化阶段被加载到内存中并同时驻留下去直到它们全部运行结束。而且，所有的应用都直接通过物理地址访问物理内存。这会带来以下问题：

- 首先，内核提供给应用的内存访问接口不够透明，也不好用。由于应用直接访问物理内存，这需要它在构建的时候就需要规划自己需要被加载到哪个地址运行。为了避免冲突可能还需要应用的开发者们对此进行协商，这显然是一件在今天看来不可理喻且极端麻烦的事情。
- 其次，内核并没有对应用的访存行为进行任何保护措施，每个应用都有整块物理内存的读写权力。即使应用被限制在 U 特权级下运行，它还是能够造成很多麻烦：比如它可以读写其他应用的数据来窃取信息或者破坏它的正常运行；甚至它还可以修改内核的代码段来替换掉原本的 ``trap_handler`` 来挟持内核执行恶意代码。总之，这造成系统既不安全、也不稳定。
- 再次，目前应用的内存使用空间在其运行前已经限定死了，内核不能灵活地给应用程序提供的运行时动态可用内存空间。比如一个应用结束后，这个应用所占的空间就被释放了，但这块空间无法动态地给其它还在运行的应用使用。

因此，为了防止应用胡作非为，本章将更好的管理物理内存，并提供给应用一个抽象出来的更加透明易用、也更加安全的访存接口，这就是基于分页机制的虚拟内存。站在应用程序运行的角度看，就是存在一个从“0”地址开始的非常大的可读/可写/可执行的地址空间（Address Space）。

实现地址空间的第一步就是实现分页机制，建立好虚拟内存和物理内存的页映射关系。此过程涉及硬件细节，不同的地址映射关系组合，相对比较复杂。总体而言，我们需要思考如下问题：

- 硬件中物理内存的范围是什么？
- 哪些物理内存空间需要建立页映射关系？
- 如何建立页表使能分页机制？
- 如何确保OS能够在分页机制使能前后的不同时间段中都能正常寻址和执行代码？
- 页目录表（一级）的起始地址设置在哪里？
- 二级/三级等页表的起始地址设置在哪里，需要多大空间？
- 如何设置页目录表项的内容？
- 如何设置其它页表项的内容？
- 如果要让每个任务有自己的地址空间，那每个任务是否要有自己的页表？
- 代表应用程序的任务和操作系统需要有各自的页表吗？
- 在有了页表之后，任务和操作系统之间应该如何传递数据？

如果能解决上述问题，我们就能设计实现具有超强防护能力的侏罗纪“头甲龙”操作系统。并可更好地理解地址空间，虚拟地址等操作系统的抽象概念与操作系统的虚存具体实现之间的联系。

..
  chyyuu：在哪里讲解虚存的设计与实现？？？


实践体验
-----------------------

本章的应用和上一章相同，只不过由于内核提供给应用的访存接口被替换，应用的构建方式发生了变化，这方面在下面会深入介绍。
因此应用运行起来的效果与上一章是一致的。

获取本章代码：

.. code-block:: console

   $ git checkout ch4

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ cd os
   $ make run

将 Maix 系列开发板连接到 PC，并在上面运行本章代码：

.. code-block:: console

   $ cd os
   $ make run BOARD=k210

如果顺利的话，我们将看到和上一章相同的运行结果（以 K210 平台为例）：

.. code-block::

   [rustsbi] RustSBI version 0.1.1
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

   [rustsbi] Platform: K210 (Version 0.1.0)
   [rustsbi] misa: RV64ACDFIMSU
   [rustsbi] mideleg: 0x22
   [rustsbi] medeleg: 0x1ab
   [rustsbi] Kernel entry: 0x80020000
   [kernel] Hello, world!
   .text [0x80020000, 0x8002b000)
   .rodata [0x8002b000, 0x8002e000)
   .data [0x8002e000, 0x8004c000)
   .bss [0x8004c000, 0x8035d000)
   mapping .text section
   mapping .rodata section
   mapping .data section
   mapping .bss section
   mapping physical memory
   [kernel] back to world!
   remap_test passed!
   init TASK_MANAGER
   num_app = 4
   power_3 [10000/300000power_5 [10000/210000]
   power_5 [20000/210000]
   power_5 [30000/210000]
   
   ...
   
   (mod 998244353)
   Test power_7 OK!
   [kernel] Application exited with code 0
   power_3 [290000/300000]
   power_3 [300000/300000]
   3^300000 = 612461288(mod 998244353)
   Test power_3 OK!
   [kernel] Application exited with code 0
   Test sleep OK!
   [kernel] Application exited with code 0
   [kernel] Panicked at src/task/mod.rs:112 All applications completed!
   [rustsbi] reset triggered! todo: shutdown all harts on k210; program halt. Type: 0, reason: 0

本章代码树
-----------------------------------------------------

.. code-block::
  :linenos:
  :emphasize-lines: 56

  ./os/src
  Rust        22 Files    1334 Lines
  Assembly     3 Files      88 Lines

  ├── bootloader
  │   ├── rustsbi-k210.bin
  │   └── rustsbi-qemu.bin
  ├── LICENSE
  ├── os
  │   ├── build.rs
  │   ├── Cargo.lock
  │   ├── Cargo.toml
  │   ├── Makefile
  │   └── src
  │       ├── config.rs(修改：新增一些内存管理的相关配置)
  │       ├── console.rs
  │       ├── entry.asm
  │       ├── lang_items.rs
  │       ├── link_app.S
  │       ├── linker-k210.ld(修改：将跳板页引入内存布局)
  │       ├── linker-qemu.ld(修改：将跳板页引入内存布局)
  │       ├── loader.rs(修改：仅保留获取应用数量和数据的功能)
  │       ├── main.rs(修改)
  │       ├── mm(新增：内存管理的 mm 子模块)
  │       │   ├── address.rs(物理/虚拟 地址/页号的 Rust 抽象)
  │       │   ├── frame_allocator.rs(物理页帧分配器)
  │       │   ├── heap_allocator.rs(内核动态内存分配器)
  │       │   ├── memory_set.rs(引入地址空间 MemorySet 及逻辑段 MemoryArea 等)
  │       │   ├── mod.rs(定义了 mm 模块初始化方法 init)
  │       │   └── page_table.rs(多级页表抽象 PageTable 以及其他内容)
  │       ├── sbi.rs
  │       ├── syscall
  │       │   ├── fs.rs(修改：基于地址空间的 sys_write 实现)
  │       │   ├── mod.rs
  │       │   └── process.rs
  │       ├── task
  │       │   ├── context.rs(修改：构造一个跳转到不同位置的初始任务上下文)
  │       │   ├── mod.rs(修改，详见文档)
  │       │   ├── switch.rs
  │       │   ├── switch.S
  │       │   └── task.rs(修改，详见文档)
  │       ├── timer.rs
  │       └── trap
  │           ├── context.rs(修改：在 Trap 上下文中加入了更多内容)
  │           ├── mod.rs(修改：基于地址空间修改了 Trap 机制，详见文档)
  │           └── trap.S(修改：基于地址空间修改了 Trap 上下文保存与恢复汇编代码)
  ├── README.md
  ├── rust-toolchain
  ├── tools
  │   ├── kflash.py
  │   ├── LICENSE
  │   ├── package.json
  │   ├── README.rst
  │   └── setup.py
  └── user
      ├── build.py(移除)
      ├── Cargo.toml
      ├── Makefile
      └── src
          ├── bin
          │   ├── 00power_3.rs
          │   ├── 01power_5.rs
          │   ├── 02power_7.rs
          │   └── 03sleep.rs
          ├── console.rs
          ├── lang_items.rs
          ├── lib.rs
          ├── linker.ld(修改：将所有应用放在各自地址空间中固定的位置)
          └── syscall.rs



本章代码导读
-----------------------------------------------------

本章涉及的代码量相对多了起来。新增的代码主要是集中在页表的处理上的。由于课程整改，春季学期的同学们可能还没有上过计组，对页表的内容还不太熟悉。因此本章的内容可能需要同学们多多回顾OS课上对页表的讲解。同时本章也会介绍我们OS的Riscv-64指令集是如何设计页表，以及页表读取和修改的方式。