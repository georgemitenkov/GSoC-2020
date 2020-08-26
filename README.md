# SPIR-V to LLVM IR dialect conversion in MLIR

A final summary of my Google Summer of Code 2020 experience with MLIR under
LLVM Compiler Infrastructure.

## Background and motivation

SPIR-V is a cross-vendor and cross-API intermediate representation that serves
both graphics and compute. In MLIR, it is modelled as a corresponding dialect
(mostly focusing on Vulkan compute), and supports multiple conversion lowerings
to it.

The aim of my project is to create a conversion path from SPIR-V dialect to LLVM
dialect. The main motivation is that this conversion enables us to embrace
SPIR-V into LLVM ecosystem via MLIR dialect conversion interface. This allows to
translate SPIR-V into CPU machine code and JIT-compile it. More practical
benefits are outlined in the proposal that can be found under `proposals`
directory in this repository.

## Patches and the work done

This section describes the patches I have submitted during the project, as well
as mentions noteworthy discussions within the community that I have participated
in. The patches are grouped logically, and are based on the common
functionality or features. All patches have been commited and pushed to master
unless stated otherwise.

1. Setting up the core infrastructure required for the SPIR-V to LLVM dialect conversion.
   Patches:
   - https://reviews.llvm.org/D81100

2. Main conversion patterns for scalar ops
   These patches include implementations and tests for arithmetic, bitwise,
   cast, comparison, and logical ops.
   Patches:
   - https://reviews.llvm.org/D81305
   - https://reviews.llvm.org/D81487
   - https://reviews.llvm.org/D81546
   - https://reviews.llvm.org/D81812
   - https://reviews.llvm.org/D82285
   - https://reviews.llvm.org/D82286
   - https://reviews.llvm.org/D82637
   - https://reviews.llvm.org/D82748
   - https://reviews.llvm.org/D82640

3. Extra type conversions 
Since SPIR-V reuses Standard dialect types, there was no need initially to add
support for scalar or vector types conversion to the `LLVMTypeConverter`. Later,
to support structs, arrays, runtime arrays and pointers additional patterns were
added.
   Patches:
   - https://reviews.llvm.org/D83285
   - https://reviews.llvm.org/D83399
   - https://reviews.llvm.org/D83403

4. SPIR-V function and module conversions
   These patches allow to convert `spv.func` and its control attributes, return
   ops and include a basic conversion of `spv.module` op.
   Patches:
   - https://reviews.llvm.org/D81931
   - https://reviews.llvm.org/D82468
   - https://reviews.llvm.org/D83786 

5. Control flow ops conversion
   This patches implement conversions for branches, function calls and
   structured control flow.
   Patches:
   - https://reviews.llvm.org/D83030
   - https://reviews.llvm.org/D83784
   - https://reviews.llvm.org/D83860
   - https://reviews.llvm.org/D83658
   - https://reviews.llvm.org/D84657
   - https://reviews.llvm.org/D84245

6. Other ops conversions
   These patches include conversion for ops that do not fall into any category
   (like `spv.Undef` for example).
   Patches:
   - https://reviews.llvm.org/D82936
   - https://reviews.llvm.org/D83291

7. Memory related ops conversion
   Patches:
   - https://reviews.llvm.org/D84224
   - https://reviews.llvm.org/D84236
   - https://reviews.llvm.org/D84396
   - https://reviews.llvm.org/D84739
   - https://reviews.llvm.org/D84626

6. GLSL ops conversion
   This patches introduce conversion patterns for ops from
   [GLSL extended instruction set](https://www.khronos.org/registry/spir-v/specs/1.0/GLSL.std.450.html)
   Patches:
   - https://reviews.llvm.org/D84627
   - https://reviews.llvm.org/D84633
   - https://reviews.llvm.org/D84661


7. Spirv runner patches
   Patches:
   - https://reviews.llvm.org/D86109
   - https://reviews.llvm.org/D86285
   - https://reviews.llvm.org/D86288
   - https://reviews.llvm.org/D86386 (Under review)
   - https://reviews.llvm.org/D86384 (Under review)
   - https://reviews.llvm.org/D86515 (Under review)
   - https://reviews.llvm.org/D86112 (Under review)
   - https://reviews.llvm.org/D86108 (Under review)
   Related discussions:
   - [Nested modules translation and symbol table](https://llvm.discourse.group/t/nested-modules-translation-and-symbol-table/1536)
   - [[RFC] mlir-spirv-runner](https://llvm.discourse.group/t/rfc-mlir-spirv-runner/1564/5)
   - [[RFC] Executing multiple MLIR modules](https://llvm.discourse.group/t/rfc-executing-multiple-mlir-modules/1616/3)

8. Documentation and style fixes
   This patches contain cthe conversion's documentation updates, as well as
   some style/bug fixes.
   Patches:
   - https://reviews.llvm.org/D83322
   - https://reviews.llvm.org/D85181
   - https://reviews.llvm.org/D85206
   - https://reviews.llvm.org/D84734
   - https://reviews.llvm.org/D85277

I have also submitted a couple of patches outside the scope of the project.
These, for example, include improving SPIR-V documentation, fixing bugs or
adding support for particular operations.
- https://reviews.llvm.org/D83791
- https://reviews.llvm.org/D84196
- https://reviews.llvm.org/D84731
- https://reviews.llvm.org/D84175

## Results

TODO: What is supported

TODO: The runner

## Important links

Original proposal can be found in this repository under `proposal` directory. In
addition, an online public version can be found [here](https://llvm.discourse.group/t/gsoc-proposal-spir-v-to-llvm-ir-dialect-conversion-in-mlir/812).

During my project, I was keeping a Google doc where I interacted with my mentors
and documented what has been done, what challenges I have encountered, and my
plans. The public copy is availiable (TODO: add link) [here]().

I have also created a manual to document what is supported in the SPIR-V to LLVM
dialect conversion at the moment. This can be found on official MLIR website
under [SPIR-V Dialect to LLVM Dialect conversion manual](https://mlir.llvm.org/docs/SPIRVToLLVMDialectConversion/).

All conversion code can be found in `SPIRVToLLVM` directory in LLVM
repository, particularly:
 - Conversion headers and implementation can be found
  [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/include/mlir/Conversion/SPIRVToLLVM)
  and [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/lib/Conversion/SPIRVToLLVM)
 - Tests are located [here](https://github.com/llvm/llvm-project/tree/05777ab941063192b9ccb1775358a83a2700ccc1/mlir/test/Conversion/SPIRVToLLVM)
 - TODO: add code location for the runner

## Future work

TODO

## Acknowledgement

TODO