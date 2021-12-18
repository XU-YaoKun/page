---
layout: post
title:  "Spider Monkey"
date:   2021-07-25 18:05:55 +0800
image:  road.jpg
tags:   [Translation]
---

# SpiderMonkey

*SpiderMonkey*是火狐浏览器的*JavaScript*和*WebAssembly*的实现库。其行为遵从[ECMAScript](https://tc39.es/ecma262/)和[WebAsssembly](https://webassembly.org/)标准。

SpiderMonkey引擎的大部分内部实现文档在源代码的注释中以[[SMDOC]](https://searchfox.org/mozilla-central/search?q=[SMDOC]&path=js%2F)的形式被标注出来了。一些有关的信息也可以在[https://spidermonkey.dev]( https://spidermonkey.dev)中找到。

## SpiderMonkey的构成

### 垃圾回收器

*JavaScript*是一种依赖垃圾回收机制的语言。在*SpiderMonkey*中，我们管理了一块堆内存，用于存放对象，以及对其中的对象进行垃圾回收。这块内存中的对象都继承自C++类型[gc::Cell](https://searchfox.org/mozilla-central/source/js/src/gc/Cell.h#109)。在每一轮垃圾回收中，任何没有被根`Cell`或者其他有效`Cell`引用的`Cell`对象都会被释放掉。

更多细节请查看[GC overview](https://firefox-source-docs.mozilla.org/js/gc.html)。

### `JS::Value`和`JS::Object`

`JavaScript`的值类型分为对象类型（Objects）和基础类型（`Undefined`，`Null`，`Boolean`，`Number`，`BigInt`，`String`，以及`Symbol`）。值对应的C++类型是[JS::Value](https://searchfox.org/mozilla-central/source/js/public/Value.h#347)，`JS::Value`可以指向一个继承自[JSObject](https://searchfox.org/mozilla-central/source/js/src/vm/JSObject.h#75)类型的对象。对象包括 1. 平常的`JavaScript`对象 2. 函数、`ArrayBuffers`、`HTML Elements`等非平常对象。

大部分的对象类型继承自`NativeObject`（`JSObject`的一个子类），`NativeObject`像哈希表一样把对象的属性以键值对的形式储存起来。这些对象类型持有属性值，并且指向一个`Shape`对象，这个`Shape`对象表示属性名的集合。相似的对象指向同一个`Shape`对象以节省内存，并且使得*JITs*可以快速地处理之前见过的对象类型。更多细节请查看[[SMDOC]Shapes](https://searchfox.org/mozilla-central/search?q=[SMDOC]+Shapes)注释。

在C++（以及Rust）代码中，可以用**JSAPI**来创建并操纵这些JavaScript对象。

### JavaScript Parser

为了能够执行脚本代码，我们首先用*Parser*将其解析为[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)（AST），然后运行*BytecodeEmitter*生成[Bytecode](https://en.wikipedia.org/wiki/Bytecode)和有关的元数据。我们将这种最终生成的格式称为[Stencil](https://searchfox.org/mozilla-central/source/js/src/frontend/Stencil.h#75)，其拥有一些良好的特性使得它不需要使用垃圾回收机制。*Stencil*可以被实例化成一系列的*GC Cells*，这些*Cells*可以被执行引擎修改和理解。

每个函数以及不包含在函数中的代码都会生成一个单独的脚本。这是以执行单元为粒度划分的，因为函数可能会被设置为回调，在之后的某个时间运行。脚本的类型有 1. `ScriptStencil` 2. `js::BaseScript` 两种。

通常来说，*parser*运行在*lazy*模式下，避免为当前脚本下的所有函数生成字节码。但是这种“懒惰”的解析方式仍然需要检查语言标准中描述的语法错误以及其他早期错误。当某个被“懒惰”解析的函数第一次执行的时候，我们会用一种叫做*delazification*的过程重新编译这段函数。“懒惰”解析避免了生成*AST*和字节码，这同时节省了`CPU`时间以及内存消耗。实际上，在加载一个网页时，其中的很多函数永远都不会执行，所以这种延迟解析的模式是非常有利的。

### JavaScript Interpreter

parser生成的*bytecode*会被用C++实现的解释器执行，解释器可以操纵GC堆内存中的对象以及调用宿主(例如：浏览器)的C++代码。[[SMDOC]Bytecode定义](https://searchfox.org/mozilla-central/search?q=[SMDOC]+Bytecode+Definitions&path=js%2F)中有对每个bytecode操作码的说明，`js/src/vm/Interpreter.cpp`有对应的实现。

### JavaScript JITs

为了加快bytecode的执行速度，我们使用了一系列的Just-In-Time（JIT）编译器，为正在运行的JavaScript代码以及正在被处理的数据生成相应平台的机器码（例如：x86，ARM等）。

当一个单独的脚本运行了很多次（或者有一个循环执行了很多次），我们就称其为热点代码，超出一定阈值之后，我们会使用JIT编译对应的热点代码。JIT会对不同程度的热点代码进行不同程度的优化。每个后续的JIT编译层会花费更多的时间在编译上，以得到更高性能的执行性能。

#### Baseline Interpreter

基础解释器是解释器和JIT的混合，每次解释执行bytecode中的一个操作码，同时将一小段叫做内联缓存（Inline Caches）的代码附着在对应的操作码上，如果接下来处理的数据是足够相似的，则解释器可以更快地运行这个操作码。更多细节请参考[[SMDOC]JIT内联缓存](https://searchfox.org/mozilla-central/search?q=[SMDOC]+JIT+Inline+Caches)注释。

#### Baseline Compiler

基础编译器采用同样的内联缓存机制，但是将整个bytecode编译成为了原生的机器码。这种策略消除了解释执行带来的调度损耗，并且对机器码进行了简单的局部优化。编译生成的机器码仍然需要调用C++代码来完成一些复杂的操作。从C++代码到机器码的过程相对较快，但是`BaselineScript`会占用内存，需要使用`mprotect`给内存设置访问权限，并且会刷新CPU缓存。

#### WarpMonkey

*WarpMonkey JIT*替代了之前的*IonMonkey*引擎，对于最高频率执行的脚本执行最高层次的优化。它能够基于数据和函数参数内联其他的脚本和代码。

我们将bytecode和内联缓存翻译成一种中间表示形式（Mid-level [Intermediate Representation](https://en.wikipedia.org/wiki/Intermediate_representation)）。中间表示形式被翻译成更接近机器的表示形式（Low-level intermediate Representation，LIR）之前，会被转化和优化。LIR执行寄存器分配的操作，然后在*Code Generation*过程中生成原生的机器码。

这里的优化假设一个脚本将要处理的数据与之前处理过的数据类似。基础的解释器生成与已执行的数据相匹配的内联缓存，这是*WrapMonkey*成功执行优化的基础。如果一个脚本被*WrapMonkey*编译之后，遇到了不匹配的数据类型，就会执行*bailout*机制。*bailout*机制重构出基础解释器的栈帧，然后跳转回基础解释器的执行路径，好像一直在执行基础解释器，没有进行后续的优化一样。

### WebAssembly

除了`JavaScript`之外，*SpiderMonkey*引擎还可以执行*WebAssembly*（WASM）代码。

#### WASM-Baseline(RabaldMonkey)

为了降低第一次执行的时延，*RabaldMonkey*引擎可以快速地将*WebAssembly*代码转化成机器码。

#### WASM-Ion(BaldrMonkey)

*BaldrMonkey*将*WASM*输入翻译成一种*MIR*形式，提供给*WarpMonkey*，并利用*IonBackend*进行优化。这些优化（特别是对寄存器的分配）生成非常快的原生机器码。

#### Cranelift

这个用于替代`BaldrMonkey`的编译器还处于试验阶段，是用`Rust`实现的优化*WASM*的编译器。目前*Cranelift*被用在基于ARM-64的平台上（这些平台不支持*BaldrMonkey*）。

