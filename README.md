# SPIR-V to LLVM IR dialect conversion in MLIR

A final summary of my Google Summer of Code 2020 experience with MLIR under
LLVM Compiler Infrastructure.

  - [Background](#background)
  - [Motivation](#motivation)
  - [Aims](#aims)
  - [Results](#results)
  - [Challenges](#challenges)
  - [Patches and the work done](#patches-and-the-work-done)
  - [Important links](#important-links)
  - [Future work](#future-work)
  - [Acknowledgement](#acknowledgement)

## Background

Instead of a single intermediate representation (IR) with a closed set of
operations and types [MLIR][MLIR] uses dialects - different flavours of IR that
form a grouping of operations and types under some common functionality. These
dialects can be converted one to another in a progressive way, as well as
translated to outside-MLIR IRs such as LLVM IR or SPIR-V.

My project particularly focuses on two dialects: SPIR-V and LLVM dialects, and
aims at implementing a missing conversion path from SPIR-V dialect to LLVM
dialect.

## Motivation

The motivation behind this project is that the SPIR-V to LLVM conversion
enables us to embrace SPIR-V into LLVM's ecosystem via MLIR dialect conversion
interface. As a result, we can convert SPIR-V dialect to LLVM, then into CPU
machine code and JIT-compile it. Also, the SPIR-V to LLVM conversion path helps
with performance checks when designing new conversions or benchmarking execution
on different hardware.

More practical benefits include supporting [SwiftShader][SwiftShader], a
CPU-based Vulkan implementation, as well as LLVM-based GPU hardware driver
compilers such as [AMDVLK][AMDVLK].

## Aims

In my proposal I have originally outlined the following aims for the project:

- Support commonly used types: scalars, vectors, arrays, pointers and structs.

- Support SPIR-V scalar operations conversion (*e.g.* arithmetic or bitwise operations).

- Support operations from [GLSL extended instruction set][GLSL].

- Support important operations such as `spv.func` and `spv.module`, as well as
SPIR-V's control flow.

- Model SPIR-V specific operations such as entry points or specialization
constants in LLVM dialect.

I have also added a stretch goal - `mlir-spirv-cpu-runner` - a tool that would
allow executing the host code and a GPU kernel fully on CPU via the conversion
path that I would have implemented.

## Results

By the end of my Google Summer of Code project, I have achieved the
following results.

- **Conversion coverage**

  In terms of the conversion coverage, the SPIR-V to LLVM conversion fully
  supports nearly all scalar and GLSL operations, all control flow operations
  (without loop and selection control), and SPIR-V functions and modules. It also
  supports all common types that were outlined in the [Aims](#aims)  section.
  
  To indicate more precise information of how the conversion works for various types and
  operations, I have created a [conversion manual][manual]. This document also describes
  any limitations of the current conversion, and types and operations that have not been
  implemented yet.

- **`mlir-spirv-cpu-runner` prototype**

  During my project, I have realised that the conversion for some specific
  SPIR-V operations may not be relevant for LLVM. For example, specialization
  constants (`spv.specConstant` in the SPIR-V dialect) are used to inject
  constant values to half-compiled shader code prior to the final compilation
  stage, and therefore are not fully relevant to the conversion flow. Similarly,
  in the CPU world a program's entry point is a "main" function. Hence,
  `spv.EntryPoint` operation conversion is mostly important for keeping the
  metadata associated with the given entry-point function (*e.g.* the workgroup
  size) in the kernel.

  As a result, my mentors and I have decided that it would be more reasonable to
  implement `mlir-spirv-cpu-runner` (originally my stretch goal).
  
  I have created a prototype of this runner tool, and all necessary passes for
  it. Please note that the patches for `mlir-spirv-cpu-runner` have not been
  committed and pushed to the main repository as for now.

  There is no multi-threading/parallelism involved in the conversion and
  therefore a case of a single-threaded GPU kernel with scalar code is
  considered. Currently, the pipeline for `mlir-spirv-cpu-runner` can be
  described as follows:
  
  - Convert the GPU kernel into SPIR-V dialect and apply all necessary
    transformations in order to get a valid SPIR-V module.

  - Emulate the kernel call by converting the launching operation into a normal
    function call. The data from the host side to the device is passed via
    copying to global variables. These are created in both the host and the
    kernel code and later linked when nested modules are folded.
  
  - Convert SPIR-V to LLVM via the new conversion path.

  ![runner's pipeline][pipeline] TODO

  After these passes the IR transforms into a nested LLVM module - a main one
  that represents the host code and a kernel module. These modules are linked
  and executed.
  
## Challenges

There were a number of challenges that I have encountered while working on the
project.

- LLVM and MLIR have a wide range of APIs that help working with data
  structures, etc. Sometimes it was not that easy to find the one that suits
  the best in a particular situation. However, thanks to advices from the
  community this problem was usually promptly resolved.

- MLIR is a young and rapidly evolving project, and I found that sometimes the
  current infrastructure may not be enough.

  Particularly, this was the case of `mlir-spirv-cpu-runner`. After all
  transformations have been applied, the IR became a nested LLVM dialect module
  that had to be translated to proper LLVM IR and linked with the nested modules
  inside it. Unfortunately, the current infrastructure of the MLIR's `JitRunner`
  and `ExecutionEngine` did not support the translation and the linking of
  multiple MLIR modules.

  The intermediate solution I have implemented was to pass an optional function
  callback to a custom LLVM IR module builder. This allowed to fold the nested
  MLIR module into a single LLVM IR module. This is not a final solution as
  `mlir::JitRunner` and `mlir::ExecutionEngine` may need to be refined to get
  closer to or reuse the functionaity of their LLVM counterparts).

## Patches and the work done

I was working on the project incrementally, submitting patches separately for
each group of operations or feature. Below is the list of patches that I have
submitted during the project, as well some noteworthy discussions within the
community that I have participated in. I decided to group patches logically -
they are based on the common functionality or features. All patches have been
commited and pushed to the main LLVM repository unless stated otherwise.

1. **Setting up the core infrastructure required for the SPIR-V to LLVM dialect conversion**

   Patches:

   - [[MLIR][SPIRVToLLVM] Add skeleton for SPIR-V to LLVM dialect conversion](https://reviews.llvm.org/D81100)

2. **Main conversion patterns for scalar operations**

   These patches include implementations and tests for arithmetic, bitwise,
   cast, comparison, and logical operationss.

   Patches:

   - [[MLIR][SPIRVToLLVM] Implemented conversion for arithmetic ops and 3 bitwise ops](https://reviews.llvm.org/D81305)
   - [[MLIR][SPIRVToLLVM] Added conversion for SPIR-V comparison ops](https://reviews.llvm.org/D81487)
   - [[MLIR][SPIRVToLLVM] Implemented shift conversion pattern](https://reviews.llvm.org/D81546)
   - [[MLIR][SPIRVToLLVM] Implemented cast ops conversion, added 4 logical ops and support of UModOp](https://reviews.llvm.org/D81812)
   - [[MLIR][LLVMDialect] Added bitreverse and ctpop intrinsics](https://reviews.llvm.org/D82285)
   - [[MLIR][SPIRVToLLVM] Conversion for bitrverse and bitcount ops](https://reviews.llvm.org/D82286)
   - [[MLIR][SPIRVToLLVM] Implementation of bitwise and logical not](https://reviews.llvm.org/D82637)
   - [[MLIR][SPIRVToLLVM] Added Bitcast conversion pattern](https://reviews.llvm.org/D82748)
   - [[MLIR][SPIRVToLLVM] Implementation of spv.BitFieldSExtract and spv.BitFieldUExtract patterns](https://reviews.llvm.org/D82640)

3. **Extra type conversions**

   Since SPIR-V reuses Standard dialect types, there was no need initially to
   add support for scalar or vector types conversion to the `LLVMTypeConverter`.
   Later, to support structs, arrays, runtime arrays and pointers additional
   patterns were added.

   Patches:

   - [[MLIR][SPIRVToLLVM] SPIR-V types size in bytes function](https://reviews.llvm.org/D83285)
   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V array, runtime array, and pointer types](https://reviews.llvm.org/D83399)
   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V struct type without offset](https://reviews.llvm.org/D83403)

4. **SPIR-V function and module conversions**

   These patches allow to convert `spv.func` and its control attributes, return
   operations and include a basic conversion of `spv.module` operation.

   Patches:

   - [[MLIR][SPIRVToLLVM] Implementation of spv.func conversion, and return ops](https://reviews.llvm.org/D81931)
   - [[MLIR][SPIRVToLLVM] Implementation of SPIR-V module conversion pattern](https://reviews.llvm.org/D82468)
   - [[MLIR][SPIRVToLLVM] SPIRV function fix and nits](https://reviews.llvm.org/D83786)

5. **Control flow operations conversion**

   These patches implement conversions for branch, function call and structured
   control flow operations.

   Patches:

   - [[MLIR][SPIRVToLLVM] SPIR-V function call conversion pattern](https://reviews.llvm.org/D83030)
   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V branch ops](https://reviews.llvm.org/D83784)
   - [[MLIR][SPIRVToLLVM] SelectionOp conversion pattern](https://reviews.llvm.org/D83860)
   - [[MLIR][LLVMDialect] Added branch weights attribute to CondBrOp](https://reviews.llvm.org/D83658)
   - [[MLIR][SPIRVToLLVM] Branch weights support for BranchConditional conversion](https://reviews.llvm.org/D84657)
   - [[MLIR][SPIRVToLLVM] Conversion pattern for loop op](https://reviews.llvm.org/D84245)

6. **Memory related operations conversion**
   
   A number of patches that implement conversion patterns for `spv.Load`,
   `spv.Variable` and other operations that involve memory handling.

   Patches:

   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V variable op](https://reviews.llvm.org/D84224)
   - [[MLIR][SPIRVToLLVM] Conversion of load and store SPIR-V ops](https://reviews.llvm.org/D84236)
   - [[MLIR][LLVMDialect] Added volatile and nontemporal attributes to load and store](https://reviews.llvm.org/D84396)
   - [[MLIR][SPIRVToLLVM] Added support of volatile and nontemporal memory access in load/store](https://reviews.llvm.org/D84739)
   - [[MLIR][SPIRVToLLVM] Conversion for global and addressof](https://reviews.llvm.org/D84626)

7. **GLSL operations conversion**

   These patches introduce conversion patterns for operations from
   [GLSL extended instruction set](GLSL).

   Patches:

   - [[MLIR][SPIRVToLLVM] Conversion patterns for GLSL ops](https://reviews.llvm.org/D84627)
   - [[MLIR][SPIRVToLLVM] Conversion for inverse sqrt and tanh](https://reviews.llvm.org/D84633)
   - [[MLIR][SPIRVToLLVM] Conversion of GLSL ops to LLVM intrinsics](https://reviews.llvm.org/D84661)

8. **Other operations conversions**

   These patches include conversion for operations that do not fall into other
   categoris (like `spv.constant` or `spv.Undef` for example).

   Patches:

   - [[MLIR][SPIRVToLLVM] Added spv.constant conversion pattern for scalar and vector types](https://reviews.llvm.org/D82936)
   - [[MLIR][SPIRVToLLVM] Miscellaneous ops conversion: select, fmul and undef](https://reviews.llvm.org/D83291)

9. **mlir-spirv-cpu-runner patches**

   In order to implement `mlir-spirv-cpu-runner`, I had to submit extra
   patches to deal with `spv.EntryPoint` or `spv.AccessChain` conversion,
   support array strides and struct offsets, as well as to create a pass
   that emulates the GPU kernel call in LLVM dialect.

   Patches:

   - [[MLIR][SPIRVToLLVM] Additional conversions for spirv-runner](https://reviews.llvm.org/D86109)
   - [[MLIR][SPIRVToLLVM] Removed std to llvm patterns from the conversion](https://reviews.llvm.org/D86285)
   - [[MLIR][SPIRV] Added optional name to SPIR-V module](https://reviews.llvm.org/D86386)
   - [[MLIR][GPUToSPIRV] Passing gpu module name to SPIR-V module](https://reviews.llvm.org/D86384)
   - [[MLIR][SPIRVToLLVM] Added a hook for descriptor set / binding encoding](https://reviews.llvm.org/D86515)
   - [[MLIR][mlir-spirv-cpu-runner] A pass to emulate a call to kernel in LLVM](https://reviews.llvm.org/D86112) (Under review)
   - [[MLIR][mlir-spirv-cpu-runner] A SPIR-V cpu runner prototype](https://reviews.llvm.org/D86108) (Under review)

   Related discussions:

   - [Nested modules translation and symbol table](https://llvm.discourse.group/t/nested-modules-translation-and-symbol-table/1536)
   - [[RFC] mlir-spirv-runner](https://llvm.discourse.group/t/rfc-mlir-spirv-runner/1564/5)
   - [[RFC] Executing multiple MLIR modules](https://llvm.discourse.group/t/rfc-executing-multiple-mlir-modules/1616/3)

10. **Documentation and style fixes**

    These patches contain the conversion's manual updates, as well as
    some style/bug fixes.

    Patches:

    - [[MLIR][SPIRVToLLVM] Documentation for SPIR-V to LLVM conversion](https://reviews.llvm.org/D83322)
    - [[MLIR][SPIRVToLLVM] Indentation and style fix in tests](https://reviews.llvm.org/D85181)
    - [[MLIR][SPIRVToLLVM] Indentation and style fix in tests](https://reviews.llvm.org/D85206)
    - [[MLIR][SPIRVToLLVM] Updated documentation for SPIR-V to LLVM conversion](https://reviews.llvm.org/D84734)
    - [[MLIR][SPIRVToLLVM] Updated LLVM types in the documentation](https://reviews.llvm.org/D85277)
    - [[MLIR][SPIRVToLLVM] Updated the documentation for the conversion](https://reviews.llvm.org/D86288)
    - [[MLIR][SPIRVToLLVM] Updated the documentation for type conversion](https://reviews.llvm.org/D86674)

11. **Patches outside SPIR-V to LLVM conversion**

    I have also submitted a couple of patches outside the scope of the project.
    These, for example, include improving SPIR-V documentation, fixing bugs or
    adding support for particular operations.

    - [[MLIR][StdToSPIRV] Fixed a typo in ops conversion tests](https://reviews.llvm.org/D83791)
    - [[MLIR][SPIRV] Updated documentation for variableOp](https://reviews.llvm.org/D84196)
    - [[MLIR][SPIRV] Added storage class constraint on global variable](https://reviews.llvm.org/D84731)
    - [[MLIR][SPIRV] Control attributes parsing/printing for loop and selection](https://reviews.llvm.org/D84175)

## Important links

The original proposal can be found in this repository under `proposal`
directory. In addition, an online public version can be found [here][proposal].

During my project, I was keeping a document where I interacted with my mentors
and tracked what has been done, what challenges I have encountered, and my
plans. The public copy is availiable [here][doc] (The comments from the original
document have tot been saved in the copy).

I have also created a manual to document what is supported in the SPIR-V to LLVM
dialect conversion at the moment. This can be found on the official MLIR website
under [SPIR-V Dialect to LLVM Dialect conversion manual][manual].

All conversion code that I wrote can be found in `SPIRVToLLVM` directory in LLVM
repository, particularly:

 - Conversion headers and implementation can be found [here][SPIRVToLLVMImpl] and
   [here][SPIRVToLLVMIncludes]

 - Tests are located [here][SPIRVToLLVMTests]

Since the `mlir-spirv-cpu-runner`'s code has not been landed to yet, I have not
included the location of the code for it. However, this can be seen in the
related patches: https://reviews.llvm.org/D86112 and
https://reviews.llvm.org/D86108.

## Future work

Personally, I plan to continue working on the SPIR-V to LLVM dialect conversion.
However, any contributions are welcome. The work on the conversion my be
continued in the following ways:

1. **Land the `mlir-spirv-cpu-runner`**

   The current version of `mlir-spirv-cpu-runner` uses a custom function
   callback to propagate information about how to construct an LLVM IR module.
   This is not a nice way and needs a better solution. A possible approach
   would be to improve `mlir::ExecutionEngine`. More can be found in related
   [revision][https://reviews.llvm.org/D86108] and
   [discussion][exec-multiple-modules].

2. **Add more type/operations conversions or scale existing conversion patterns**

   A great way to continue the current work would be adding new conversion
   patters, *e.g.* for atomic ops. Also, more types like `spv.matrix` can be
   supported.
   
   Another possible contribution is to scale some of the existing patterns,
   including but not limited to having a `spv.constant` to support arrays and
   structs or map `spv.loop`'s control to LLVM IR metadata.
   
   Note that what has not been done can be easily deduced from the conversion
   [manual][manual] described above.

3. **Model SPIR-V decorations in LLVM dialect**

   This project did not intend to add support for SPIR-V decoration attributes.
   However, they can be mapped to LLVM IR metadata/flags/etc. A good starting
   point would be a [post][decorations] on modelling some of these decorations.

4. **Map GPU-level multi-threading/parallelism to LLVM**

   A very interesing next step is to find a way of how to represent GPU's
   workgroups, blocks and threads on the CPU level. This requires a major
   discussion within the community, so it can be considered as a long-term
   goal.

## Acknowledgement

I would like to thank my mentors Lei Zhang and Mahesh Ravishankar for their
guidance and support along the project. I have learnt a lot about MLIR's
ecosystem, SPIR-V, Vulkan and GPU programming. Also, I would like to thank
Alex Zinenko for his help on the LLVM side, and River Riddle for his help with
code reviews and C++/LLVM/MLIR APIs.

Special thanks to all LLVM/MLIR community members for their advice and comments.

[AMDVLK]: https://github.com/GPUOpen-Drivers/AMDVLK
[decorations]: https://llvm.discourse.group/t/spir-v-to-llvm-dialect-conversion-decorations-and-other-attributes-semantics/1179
[doc]: https://docs.google.com/document/d/1nnA8Rf3R18_V4GsYu0rWNFAUoH6fewFptWH08Eqx4gI/edit?usp=sharing
[exec-multiple-modules]: https://llvm.discourse.group/t/rfc-executing-multiple-mlir-modules/1616/3
[GLSL]: https://www.khronos.org/registry/spir-v/specs/1.0/GLSL.std.450.html
[manual]: https://mlir.llvm.org/docs/SPIRVToLLVMDialectConversion/
[MLIR]: https://mlir.llvm.org
[proposal]: https://llvm.discourse.group/t/gsoc-proposal-spir-v-to-llvm-ir-dialect-conversion-in-mlir/812
[SPIRVToLLVMImpl]: https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/lib/Conversion/SPIRVToLLVM
[SPIRVToLLVMIncludes]: https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/include/mlir/Conversion/SPIRVToLLVM
[SPIRVToLLVMTests]: https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/test/Conversion/SPIRVToLLVM
[SwiftShader]: https://github.com/google/swiftshader
