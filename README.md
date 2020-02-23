## Artifact Submission
 - Accepted Paper [pdf](https://github.com/sdasgup3/PLDI-2020-Artifact-Evaluation/blob/master/pldi2020-paper29.pdf)
 - VM Details
    - VM Player: VirtualBox 6.0.4 Or VirtualBox 5.1
    - Ubuntu Image: ova
      -   md5 hash: 3501fcd9f082d448c63e2a642fbc4718
      -   login: sdasgup3
      -   password: aecadmin123
    - Guest Machine requirements
    - Minimum 8 GB of RAM.
    - Recommended number of processors is 4 to allow parallel experiments.

**NOTE**: For a Ubuntu host machine with Secure Boot enabled, the presented VirtualBox image may fail to be loaded. In that case, you can either disable the Secure Boot, or sign the VirtualBox module as described [here](https://askubuntu.com/questions/900118/vboxdrv-sh-failed-modprobe-vboxdrv-failed-please-use-dmesg-to-find-out-why/900121#900121).

## Program-Level validation (PLV)
### Source Code

The two major components of PLV are [Compositional Lifter](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/decompiler/decompiler.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/compositional-decompiler/compositional-decompiler.cpp) and [matcher](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/matcher/matcher.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/llvm-graph-matching/llvm-graph-matching.cpp). Additionally, Compositional Lifter uses a [_Store_](https://github.com/sdasgup3/compd_cache) of formally validated instructions to stitch the validated LLVM IR sequence of constituent instructions in a binary program.

## Testing Arena for PLV

The [test directory](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/) contains test-suites like `toy-examples` and `single-source-benchmark`. The [single-source-benchmark](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark) contain folders for all the programs hosted by the test-suite. Each such program, for example the [Queens program](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark/Queens), has the following structure

 - Doit/Initrand/Rand/main/Try/Queens (Artifacts related to individual functions of the `Queens`'s program, extracted using [wllvm](https://github.com/travitch/whole-program-llvm))
 - src (the source artifacts of the program)
   - test.ll/test.bc (Source llvm code of `Queens`'s program) 
 - binary
   - test: Binary compiled from src/test.bc
   - test.mcsema.bc: McSema lifted LLVM IR from `binary/test`
   - test.mcsema.inline.ll: Inlined version of `binary/test.mcsema.bc`
   - test.mcsema.calls_renamed.ll: Version of `binary/test.mcsema.inline.ll` with function calls renamed so as to prevent inter-procedural optimization during normalization [1]  
   - test.reloc: Binary compiled from src/test.bc with relocation information preserved
 - Makefile: With the following relevant targets
   - binary: To generate `binary/test`
   - reloc_binary: To generate `binary/test.reloc`
   - mcsema: To generate `binary/test.mcsema.bc`,`binary/test.mcsema.inline.ll`, and `binary/test.mcsema.calls_renamed.ll`  

### Experiments
In the VM, 


