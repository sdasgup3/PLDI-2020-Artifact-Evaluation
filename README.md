## Artifact Submission
 - Accepted Paper [pdf](https://github.com/sdasgup3/PLDI-2020-Artifact-Evaluation/blob/master/pldi2020-paper29.pdf)
 - VM Details
    - VM Player: VirtualBox 6.0.4 Or VirtualBox 5.1
    - Ubuntu Image: [ova](https://drive.google.com/file/d/1DATqsXzX7mg9nq6tgKlQfl2hnTc26yFS/view?usp=sharing)
      -   md5 hash: 3501fcd9f082d448c63e2a642fbc4718
      -   login: sdasgup3
      -   password: aecadmin123
    - Guest Machine requirements
      - Minimum 8 GB of RAM.
      - Recommended number of processors is 4 to allow parallel experiments.
  - We have also included the current repository in the VM disk, so that one can access this README.md file at `~/Github/PLDI20-Artifact-Evaluation/README.md`. This is just in case the bidirectional shared clipboard does not work on the VirtualBox.
  
**Troubleshoot**: 
1. For a Ubuntu host machine with Secure Boot enabled, the presented VirtualBox image may fail to be loaded. In that case, you can either disable the Secure Boot, or sign the VirtualBox module as described [here](https://askubuntu.com/questions/900118/vboxdrv-sh-failed-modprobe-vboxdrv-failed-please-use-dmesg-to-find-out-why/900121#900121).
2. [VM bot up with black login screen](https://askubuntu.com/questions/1134892/ubuntu-18-04-lts-on-virtualbox-boots-up-but-black-login-screen)
3. Virtualbox `Error ID: BLKCACHE_IOERR`: In the VB client, Storage » SATA Controller" Use the cache I/O host (all other values are those used by default VirtualBox))

## Program-Level validation (PLV)
### Source code

The two major components of PLV are [Compositional Lifter](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/decompiler/decompiler.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/compositional-decompiler/compositional-decompiler.cpp) and [matcher](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/matcher/matcher.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/llvm-graph-matching/llvm-graph-matching.cpp). Additionally, Compositional Lifter uses a [_Store_](https://github.com/sdasgup3/compd_cache) of formally validated instructions to stitch the validated LLVM IR sequence of constituent instructions in a binary program.

### Testing arena for PLV

The [test directory](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/) contains test-suites like `toy-examples` and `single-source-benchmark`. The [single-source-benchmark](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark) contain folders for all the programs hosted by the test-suite. Each such program, for example the [Queens program](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark/Queens), has the following structure:


 - `src` (the source artifacts of the program)
   - `test.ll/test.bc` (Source LLVM code of `Queens`'s program) 
 - binary
   - `test`: Binary compiled from src/test.bc
   - `test.mcsema.bc`: McSema lifted LLVM IR from `binary/test`
   - `test.mcsema.calls_renamed.ll`: A version of `binary/test.mcsema.ll` with function calls renamed so as to prevent inter-procedural optimization during normalization [1]
   - `test.mcsema.inline.ll`: Inlined version of `binary/test.mcsema.calls_renamed.ll`
   - `test.reloc`: Binary compiled from src/test.bc with relocation information preserved
 - `Makefile`: With the following relevant targets
   - `binary`: To generate `binary/test`
   - `reloc_binary`: To generate `binary/test.reloc`
   - `mcsema`: To generate `binary/test.mcsema.bc`,`binary/test.mcsema.inline.ll`, and `binary/test.mcsema.calls_renamed.ll`  
  - `Doit (or Initrand/Rand/main/Try/Queens)` (Artifacts related to individual functions of the `Queens`'s program, extracted using [wllvm](https://github.com/travitch/whole-program-llvm))
    - `mcsema/test.proposed.ll`: The LLVM IR, corresponding to function `Doit`, generated using Compositional Lifter.
    - `mcsema/test.proposed.inlined.ll`: Inlined version of `mcsema/test.proposed.ll`
    - `Makefile`: With the following relevant targets
      - `compd`: Invoke Compositional Lifter to generate `mcsema/test.proposed.inline.ll`
      - `compd_opt`: Invoke Normalizer to normalize `mcsema/test.proposed.inline.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L15)
      - `mcsema_opt`: Invoke Normalizer to normalize `binary/test.mcsema.calls_renamed.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L15)
      - `match`: Invoke Matcher to check for graph-isomorphism on the data-dependence graphs of the above normalized versions.

**Please Note** 
 1. We have pre-populated the McSema lifted LLVM IR `<program name>/binary/test.mcsema.bc` because McSema needs a licensed disassembler `IDA` to generate this file, which is not provided because of some licensing issues.
 2. The [_Store_](https://github.com/sdasgup3/compd_cache) has the validated IR sequences of individual binary instructions,  generated using McSema. As a result, we have packaged the entire _Store_ in the VM, so that the reviewer do not have to run McSema. 
 
### Running the PLV pipeline

#### An example run
Here we will elaborate the process of running PLV on an isolated example function `Queens/Doit/`. 
We use shell variable NORM to specify which set of optimization passes to use for normalization. For example, the value `CUSTOM` enables using a [set of 17 LLVM opt passes](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L16) for normalization. Other values are not relevant for the current submission. 

Running PLV on it involves the following steps
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/

## Running Mcsema to lift the binary "../binary/test"
## Creates  "../binary/test.mcsema.inline.ll"
# make mcsema // Skip  this steps as it requires running mcsema + IDA.

## Running Compositional lifter on the corresponding binary ../binary/test.reloc
## Creates mcsema/test.proposed.inline.ll
make compd

## Normalizing mcsema/test.proposed.inline.ll
## Creates mcsema/test.proposed.opt.ll
make compd_opt

## Normalizing ../binary/test.mcsema.inline.ll (Already populated using make mcsema)
## Creates mcsema/test.mcsema.opt.ll
make mcsema_opt

## Matching mcsema/test.proposed.opt.ll & mcsema/test.mcsema.opt.ll
make match
```
**Please Note**
The make target `match` already includes the normalization actions of `compd_opt` & `mcsema_opt`, hence one can skip invoking the redundant targets. The reason we still have those targets are for legacy reasons and included in this presentation
for better explanation of the steps. For example. instead of the above steps one can do
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/
make compd
make match
```

 
### Batch run
To demonstrate a batch run, we have included a list of sample functions in a list `~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/samplePassList.txt`. The list also
includes function `himenobmtxpa/jacobi`, the biggest function we tried lifting before submission, wherein the size of the extracted LLVM IR, using the Compositional Lifter, is `32105` LOC.

Running PLV in batch mode involves the following steps
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/

## Running Compositional Lifter
cat docs/AE_docs/samplePassList.txt | parallel "cd {}; make  compd; cd - "

## Running Normalizer + Matcher
cat docs/AE_docs/samplePassList.txt | parallel "cd {}; make  match - "
```
**Please Note**
We have provided the list of `2189` passing cases `~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/matcherPassList.txt`, which can 
be run using either batch mode or individually.

## Single-Instruction validation (SIV)
### Source code
An important component of the SIV is the tool [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp), which converts the symbolic summary (specified in K-AST) to SMTLIB queries, with the core functionality defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/smt-generator/smt-generator.cpp).

### Testing for SIV
For each binary instruction, for example the [addq_r64_r64](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/register-variants/adcq_r64_r64), has the following structure:

 - `test.c`: C file with an instruction instance of `addq` wrapped in inline assembly.
 - `test`: Binary compiled from above C file. 
 - `test.ll`: McSema lifted LLVM IR from binary `test`.
 - `test.mod.ll`: McSema lifts a lot af glue code other than the instructions in `test`. This is de-cluttered version of `test.ll` focusing on just the IR relevant to the lifting of `addq` instruction.
 - `test-xspec.k`: The K-specification file necessary for running a symbolic execution engine `krpove` (a tool that K-framework generates automatically from the semantics of X86-64 ISA) on binary instruction and generating symbolic summary.
 - `Output/test-xspec.out`: Output of the above symbolic execution.
 - `test-lspec.k`: The K-specification file necessary for running another symbolic execution engine `kprove` (a tool that K-framework generates automatically from the semantics of LLVM IR) on `test.mod.ll` and generating symbolic summary.
 - `Output/test-lspec.out`: Output of the above symbolic execution.
 - `Output/test-z3.py`: The python file encoding the verification queries in SMTLIB format. The queries are created using the tool [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp) by processing the symbolic summaries in  `Output/test-xspec.out` and `Output/test-lspec.out`.
 - `Makefile`: With the following relevant targets
   - `binary`: Generate `test` from `test.c`.
   - `mcsema`: Lift `test` to `test.ll` using McSema.
   - `declutter`: Sanitize `test.ll` to `test.mod.ll`.
   - `genxspec`: Genearte the `test-xspec.k` file.
   - `collect`: Prepare for building the symbolic execution engine using the semantics definitions of the binary instructions included in `test`.
   - `kompile`: Generate the symbolic execution engine for binary instructions included in `test`. 
   - `kli`: Run concrete execution of `test.mod.ll` using the LLVM semantics. Output is stored in `Output/test-lstate.out`
   - `genlspec`: Generate the `test-lspec.k` file using many details from the concrete execution log `Output/test-lstate.out`.
   - `lprove`: Invoke symbolic execution using the spec file `test-lspec.k`. Generates `Output/test-lspec.out` containing the symbolic summary for the LLVM ir `test.mod.ll`.
   - `xprove`: Invoke symbolic execution using the spec file `test-xspec.k`. Generates `Output/test-xspec.out` containing the symbolic summary for the binary instruction `addq`.
   - `genz3`: Invoke [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp) to create `Output/test-z3.py` containing verification queries.
   - `provez3`: Invoke z3 on ``Output/test-z3.py.
   
**Please Note** 
  1. Similar files are available for other instructions.
  2. We have pre-populated the McSema lifted LLVM IR `<instruction opcode>/test.ll` because McSema is not included in the VM distribution.

### Running the SIV pipeline

#### An example run
Running SIV on an isolated example instruction `addq_r64_r64` involves the following step
```
~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/run_standalone.sh register-variants/addq_r64_r64
```

### Details about SIV pipeline
The above script runs the following pipeline
```
PROG=$1
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/$PROG
make declutter # Sanitize the McSema lifter IR, test.ll
make genxspec # Create spec-file for running sym-ex on binary instruction
make collect; make kompile ; # Generate sym-ex engine 
make xprove # Run symbolic execution
make kli; make genlspec # Create spec-file for running sym-ex on LLVM ir sequence
make genz3 # Generate verification queries
make provez3 # Dispatch verification queries to z3
cd -
```

One of the most interesting step is `make provez3`, responsible for running z3 on the verification queries encoded in `Output/test-z3.py` [an example](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/single_instruction_translation_validation/mcsema/register-variants/adcq_r64_r64/Output/test-z3.py). The python file has the following format:
```
s = z3.solver()
For each E in {registers, flags, memory values}:
 s.push()

 lvar = symbolic summary corresponding to E, obtained by sym-execution of the LLVM IR (generated by McSema by lifting the binary instruction.  
 xvar = symbolic summary corresponding to E, obtained by sym-execution the binary instruction.

 s.add(lvar != xvar)

 solve s for unsat/sat/timeout

 s.pop()

```
The verification queries, for each register/flag/memory value, are dispatched upon the make target `make provez3`. The output `Test-Pass` is generated if all the queries results in `unsat`. Conversely, the output says `Test-Fail` if any of query results in `sat or timeout`.

In order to run this prover step on a sample of [non-buggy instructions](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/sample-non-bugs.txt), do the following:
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/$PROG
cat docs/AE_docs/non-bugs.txt | parallel "echo {}; echo ===; cd {}; make provez3; cd -" |& tee ~/Junk/log 
```

#### Reproducing bugs
We provide the [a list of bugs](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/bugs.txt). In order to show
that the result of Z3-comparison on the corresponding verification queries, do the following:
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
cat docs/AE_docs/bugs.txt | parallel "cd {}; make provez3; cd -"
```
The above command will show, for each buggy instruction, the register has mismatching symbolic summaries.

We know, from our past experience, that having both the operands equal makes the semantics of some instructions pretty challenging. We leverage that experience by running SIV on instructions with its operands being the same register and found coupe of [bugs](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/same-reg-bugs.txt).
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
cat docs/AE_docs/same-reg-bugs.txt | parallel "cd {}; make provez3; cd -"
```

Bug Reports to McSema can be found [here](https://github.com/lifting-bits/remill/issues/376), which are all acknowledged. 

#### Reproducing timeouts
We found that [timeouts]((https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/flaky_TO.txt)) (related to paddb, psubb) are flaky: the Z3 solver results toggled between unknown and unsat depending on the order in which other unrelated constraints are added (a known z3 issue: https://github.com/Z3Prover/z3/issues/1106). By removing the unrelated constraints, z3 concludes them to be equivalent. For example, for the above flaky cases, the current version of `test-z3.py` has the unrelated constraints commented-in (and marked with `REMOVE: START` and `REMOVE: END` tags). Hence, following the steps will them all of them pass
```
cat docs/AE_docs/flaky_TO.txt | parallel "cd {}; make provez3; cd -"
```

However, in we remove the comment (marked with `REMOVE: START` and `REMOVE: END` tags), from say `psubb_xmm_xmm`, are run it again, then we will get the long timeout. Here are the steps
```
cd register-variants/psubb_xmm_xmm
// Remove the comment marked by `REMOVE: START` and `REMOVE: END` tags
make provez3
```

The remaining case (related to mulq) include solver constraints containing bit vector multiplication which the state-of-the-art SMT solvers are not very efficient at reasoning about. These cases timed-out (after 24h) reproducibly.
```
cd register-variants/mulq_r64
make provez3 // Timeout provides is 24 hrs
```
