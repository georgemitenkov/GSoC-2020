# SPIR-V to LLVM IR dialect conversion in MLIR

A final summary of my Google Summer of Code 2020 experience with MLIR under
LLVM Compiler Infrastructure.

## Background and motivation

SPIR-V is a cross-vendor and cross-API intermediate representation that serves
both graphics and compute. In MLIR, it is modelled as a corresponding dialect
(mostly focusing on Vulkan compute), and supports multiple conversion lowerings
to and from it. In my project, I created a missing conversion path from SPIR-V
dialect to LLVM dialect.

The main motivation is that this conversion enables us to embrace
SPIR-V into LLVM ecosystem via MLIR dialect conversion interface. This allows to
translate SPIR-V into CPU machine code and JIT-compile it. More practical
benefits are outlined in the proposal that can be found under `proposal`
directory in this repository.

## Results

Having been working on this project for 3 months, the following results were
achieved.

- Coverage

  The conversion nearly fully supports all scalar operations. More precise
  information on the supported types and operations can be found in the
  [conversion manual](https://mlir.llvm.org/docs/SPIRVToLLVMDialectConversion/).

- SPIR-V CPU runner prototype

  In order to evaluate the correctness of the conversion, I have
  implemented a prototype of `mlir-spirv-cpu-runner` that is currently
  under review. The main idea is that the module with Stadard dialect host
  code and a GPU dialect kernel module can be fully converted to LLVM dialect
  via SPIR-V to LLVM path.

  At the moment, there is no multi-threading/parallelism involved in the
  conversion. Hence, my mentors and I decided to limit the runner for the case
  of a single-threaded kernel. The results are still a great indicator of the
  semantically correct conversion.

## Patches and the work done

This section describes the patches I have submitted during the project, as well
as mentions noteworthy discussions within the community that I have participated
in. The patches are grouped logically, and are based on the common
functionality or features. All patches have been commited and pushed to master
unless stated otherwise.

1. **Setting up the core infrastructure required for the SPIR-V to LLVM dialect conversion**

   Patches:

   - [[MLIR][SPIRVToLLVM] Add skeleton for SPIR-V to LLVM dialect conversion](https://reviews.llvm.org/D81100)

2. **Main conversion patterns for scalar ops**

   These patches include implementations and tests for arithmetic, bitwise,
   cast, comparison, and logical ops.

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
   ops and include a basic conversion of `spv.module` op.

   Patches:

   - [[MLIR][SPIRVToLLVM] Implementation of spv.func conversion, and return ops](https://reviews.llvm.org/D81931)
   - [[MLIR][SPIRVToLLVM] Implementation of SPIR-V module conversion pattern](https://reviews.llvm.org/D82468)
   - [[MLIR][SPIRVToLLVM] SPIRV function fix and nits](https://reviews.llvm.org/D83786)

5. **Control flow ops conversion**

   These patches implement conversions for branches, function call and
   structured control flow.

   Patches:

   - [[MLIR][SPIRVToLLVM] SPIR-V function call conversion pattern](https://reviews.llvm.org/D83030)
   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V branch ops](https://reviews.llvm.org/D83784)
   - [[MLIR][SPIRVToLLVM] SelectionOp conversion pattern](https://reviews.llvm.org/D83860)
   - [[MLIR][LLVMDialect] Added branch weights attribute to CondBrOp](https://reviews.llvm.org/D83658)
   - [[MLIR][SPIRVToLLVM] Branch weights support for BranchConditional conversion](https://reviews.llvm.org/D84657)
   - [[MLIR][SPIRVToLLVM] Conversion pattern for loop op](https://reviews.llvm.org/D84245)

6. **Memory related ops conversion**
   
   A number of patches that implement conversion patterns for `spv.Load`,
   `spv.Variable` and other ops that involve memory handling.

   Patches:

   - [[MLIR][SPIRVToLLVM] Conversion of SPIR-V variable op](https://reviews.llvm.org/D84224)
   - [[MLIR][SPIRVToLLVM] Conversion of load and store SPIR-V ops](https://reviews.llvm.org/D84236)
   - [[MLIR][LLVMDialect] Added volatile and nontemporal attributes to load and store](https://reviews.llvm.org/D84396)
   - [[MLIR][SPIRVToLLVM] Added support of volatile and nontemporal memory access in load/store](https://reviews.llvm.org/D84739)
   - [[MLIR][SPIRVToLLVM] Conversion for global and addressof](https://reviews.llvm.org/D84626)

7. **GLSL ops conversion**

   These patches introduce conversion patterns for ops from
   [GLSL extended instruction set](https://www.khronos.org/registry/spir-v/specs/1.0/GLSL.std.450.html).

   Patches:

   - [[MLIR][SPIRVToLLVM] Conversion patterns for GLSL ops](https://reviews.llvm.org/D84627)
   - [[MLIR][SPIRVToLLVM] Conversion for inverse sqrt and tanh](https://reviews.llvm.org/D84633)
   - [[MLIR][SPIRVToLLVM] Conversion of GLSL ops to LLVM intrinsics](https://reviews.llvm.org/D84661)

8. **Other ops conversions**

   These patches include conversion for ops that do not fall into any category
   (like `spv.constant` or `spv.Undef` for example).

   Patches:

   - [[MLIR][SPIRVToLLVM] Added spv.constant conversion pattern for scalar and vector types](https://reviews.llvm.org/D82936)
   - [[MLIR][SPIRVToLLVM] Miscellaneous ops conversion: select, fmul and undef](https://reviews.llvm.org/D83291)

9. **mlir-spirv-cpu-runner patches**

   In order to implement `mlir-spirv-cpu-runner`, I had to submit extra
   patches to deal with `spc.EntryPoint` or `spv.AccessChain` conversion, as
   well as to support array strides and struct offsets.

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

    These patches contain the conversion's documentation updates, as well as
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

Original proposal can be found in this repository under `proposal` directory. In
addition, an online public version can be found [here](https://llvm.discourse.group/t/gsoc-proposal-spir-v-to-llvm-ir-dialect-conversion-in-mlir/812).

During my project, I was keeping a Google doc where I interacted with my mentors
and documented what has been done, what challenges I have encountered, and my
plans. The public copy is availiable [here](https://docs.google.com/document/d/1nnA8Rf3R18_V4GsYu0rWNFAUoH6fewFptWH08Eqx4gI/edit?usp=sharing).

I have also created a manual to document what is supported in the SPIR-V to LLVM
dialect conversion at the moment. This can be found on official MLIR website
under [SPIR-V Dialect to LLVM Dialect conversion manual](https://mlir.llvm.org/docs/SPIRVToLLVMDialectConversion/).

All conversion code can be found in `SPIRVToLLVM` directory in LLVM
repository, particularly:
 - Conversion headers and implementation can be found
  [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/include/mlir/Conversion/SPIRVToLLVM)
  and [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/lib/Conversion/SPIRVToLLVM)
 - Tests are located [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/test/Conversion/SPIRVToLLVM)

The runner code has not been landed to master yet.

## Future work

The work on the SPIR-V to LLVM dialect conversion my be continued in the
following ways:

1. **Land the `mlir-spirv-cpu-runner`**

   The current version of `mlir-spirv-cpu-runner` uses a custom function
   callback to propagate information about how to construct an LLVM IR module.
   This is not a nice way and needs a better solution. A possible approach
   would be to improve `mlir::ExecutionEngine`. More can be found in related
   [revision](https://reviews.llvm.org/D86108) and
   [discussion](https://llvm.discourse.group/t/rfc-executing-multiple-mlir-modules/1616/3).

2. **Add more type/op conversions or scale existing conversion patterns**

   A great way to continue the current work would be adding new conversion
   patters, *e.g.* for atomic ops. Also, more types like `spv.matrix` can be
   supported.
   
   Another possible contribution is to scale some of the existing patterns,
   including but not limited to having a `spv.constant` to support arrays and
   structs or map `spv.loop`'s control to LLVM IR metadata.
   
   Not that what has not been done can be easily deduced from the conversion
   manual described above.

3. **Model SPIR-V decorations in LLVM dialect**

   This project did not intend to add support for SPIR-V decoration attributes.
   However, they can be mapped to LLVM IR metadata/flags/etc. A good starting
   point would be a [post](https://llvm.discourse.group/t/spir-v-to-llvm-dialect-conversion-decorations-and-other-attributes-semantics/1179) on modelling some of these decorations.

4. **Map GPU-level multi-threading/parallelism to LLVM**

   A very interesing next step is to find a way how to represent GPU's
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
