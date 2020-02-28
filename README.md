## Artifact Submission
 - Accepted Paper [pdf](https://github.com/sdasgup3/PLDI-2020-Artifact-Evaluation/blob/master/pldi2020-paper29.pdf)
 - VM Details
    - VM Player: [VirtualBox](https://www.virtualbox.org/) 6.1 Or 5.1
    - Ubuntu Image: [ova](https://drive.google.com/file/d/10F57jxFIWb5R-W0JvFilZll0hzWQUiz8/view?usp=sharing)
      -   [md5 hash](https://docs.google.com/document/d/1YzOBUxWMoXes9bcqWTJYrXeWxVRGFUnefhs8jDxDabs/edit?usp=sharing)
      -   login: sdasgup3
      -   password: aecadmin123
    - Guest Machine requirements
      - Minimum 8 GB of RAM.
      - Minimum number of processors is 4 to allow parallel experiments.
      - Recommended to configure the VM with more RAM and processor to allow aggressive parallel experiments.
  - We have also included the current repository in the VM disk, so that one can access this README.md file at `~/Github/PLDI20-Artifact-Evaluation/README.md`. This is just in case the bidirectional shared clipboard does not work for the VirtualBox.
  
**Troubleshoot**: 
1. For a Ubuntu host machine with Secure Boot enabled, the presented VirtualBox image may fail to be loaded. In that case, you can either disable the Secure Boot, or sign the VirtualBox module as described [here](https://askubuntu.com/questions/900118/vboxdrv-sh-failed-modprobe-vboxdrv-failed-please-use-dmesg-to-find-out-why/900121#900121).
2. [VM boot up with black login screen](https://askubuntu.com/questions/1134892/ubuntu-18-04-lts-on-virtualbox-boots-up-but-black-login-screen)
3. Virtualbox `Error ID: BLKCACHE_IOERR`: In the VB client, Storage » SATA Controller" Use the cache I/O host (all other values are those used by default VirtualBox))

## Program-Level validation (PLV)
### Source code

The two major components of PLV are [Compositional Lifter](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/decompiler/decompiler.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/compositional-decompiler/compositional-decompiler.cpp) and [matcher](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/matcher/matcher.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/llvm-graph-matching/llvm-graph-matching.cpp). Additionally, Compositional Lifter uses a [_Store_](https://github.com/sdasgup3/compd_cache) of formally validated instructions to stitch the validated LLVM IR sequence of constituent instructions in a binary program.

### Testing arena for PLV

The [test directory](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/) contains test-suites like `toy-examples` and `single-source-benchmark`. The [single-source-benchmark](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark) contain folders for all the programs hosted by the test-suite. Each such program, for example the [Queens program](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark/Queens), has the following structure:


 - `src` (the source artifacts of the program)
   - `test.ll or test.bc` (Source LLVM code of `Queens`'s program) 
 - binary
   - `test`: Binary compiled from src/test.bc
   - `test.mcsema.bc`: McSema lifted LLVM IR from `binary/test`
   - `test.mcsema.calls_renamed.ll`: A version of `binary/test.mcsema.ll` with function calls renamed so as to prevent inter-procedural optimization during normalization
   - `test.mcsema.inline.ll`: Inlined version of `binary/test.mcsema.calls_renamed.ll`
   - `test.reloc`: Binary compiled from src/test.bc with relocation information preserved
 - `Makefile`: With the following relevant targets
   - `binary`: To generate `binary/test`
   - `reloc_binary`: To generate `binary/test.reloc`
   - `mcsema`: To generate `binary/test.mcsema.bc`,`binary/test.mcsema.inline.ll`, and `binary/test.mcsema.calls_renamed.ll` (To be Skipped)
  - `Doit (or Initrand/Rand/main/Try/Queens)` (Artifacts related to individual functions of the `Queens`'s program, extracted using external tool [wllvm](https://github.com/travitch/whole-program-llvm))
    - `mcsema/test.proposed.ll`: The LLVM IR, corresponding to function `Doit`, generated using Compositional Lifter.
    - `mcsema/test.proposed.inlined.ll`: Inlined version of `mcsema/test.proposed.ll`
    - `Makefile`: With the following relevant targets
      - `compd`: Invoke Compositional Lifter to generate `mcsema/test.proposed.inline.ll`
      - `compd_opt`: Invoke Normalizer to normalize `mcsema/test.proposed.inline.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L15). Generates `mcsema/test.proposed.opt.ll`.
      - `mcsema_opt`: Invoke Normalizer to normalize `../binary/test.mcsema.inline.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L15). Generates `mcsema/test.mcsema.opt.ll`
      - `match`: Invoke Matcher to check for graph-isomorphism on the data-dependence graphs of the above normalized versions `mcsema/test.proposed.opt.ll` & `mcsema/test.mcsema.opt.ll`.

**Note** 
 1. We have pre-populated the McSema lifted LLVM IR `<program name>/binary/test.mcsema.bc` because McSema needs a licensed disassembler `IDA` to generate this file, which is not provided because of some licensing issues.
 2. The [_Store_](https://github.com/sdasgup3/compd_cache) has the validated IR sequences of individual binary instructions,  generated using McSema. As a result, we have packaged the entire _Store_ in the VM, so that the reviewer do not have to run McSema. 
 
### Running the PLV pipeline

#### An example run
Here we will elaborate the process of running PLV on an isolated example function `Queens/Doit/`. 
We use shell variable NORM to specify which set of optimization passes to use for normalization. For example, the value `CUSTOM` enables using a [set of 17 LLVM opt passes](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/scripts/matcher_driver.sh#L16) for normalization. Another option for NORM enables autotuner for pass selection. but is relevant for the current submission. 

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
**Note**
The Make target `match` already includes the normalization actions of
`compd_opt` & `mcsema_opt`, hence one can skip invoking the redundant targets.
The reason we still have those targets are for legacy reasons and included in
this presentation for better explanation of the steps. For example. instead of
the above steps one can do
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/
make compd
make match
```

### Batch run (Recommended)
To demonstrate a batch run, we have included a list of sample functions in a
list
`~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/samplePassList.txt`.
The list also includes function `himenobmtxpa/jacobi`, the biggest function we
tried lifting before submission, wherein the size of the extracted LLVM IR,
      using the Compositional Lifter, is `32105` LOC.

Running PLV in batch mode, over a sample list of functions, involves the following steps
```
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/
../../scripts/run_batch_plv.sh docs/AE_docs/samplePassList.txt

# cat ../../scripts/run_batch_plv.sh
## LIST=$1
## 
## 
## TESTARENA=" ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/"
## 
## echo "Batch Run Begin"
## echo
## 
## echo "Setting NORMALIZATION to custom passes"
## export NORM=CUSTOM
## 
## echo
## echo "Running Compositional Lifter + Normalizer + Matcher"
## cat $LIST | parallel "echo ; echo {}; echo =======; cd {}; make  compd; make match; cd - "
## 
## echo "Batch Run Begin"
```

**Note**
We have provided the list of `2189` passing cases
`~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/matcherPassList.txt`,
  which can be run using either batch mode or individually.

The reviewer is encouraged to pick a random count of entries form the file and invoke the batch run
```
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/
sort -R docs/AE_docs/matcherPassList.txt | head -n 10 > /dev/stdout | ../../scripts/run_batch_plv.sh   /dev/stdin
```

### PLV injected bug detection
We provided a [list of
test-cases](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/sampleInjectedBugs.txt),
to demonstrate the effectiveness of PLV to catch potential bugs. The test-cases
represent the four different category of injections as mentioned in paper (line
    1073) and generated by modifying the McSema-lifted-IR function
`Queens/Rand`.

For example, an example injection can be found at
`~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Inject.1/binary/test.mcsema.ll`,
  where the bug is marked with comment tag `INJECTED_BUG`, along with the original line
  of code marked by tag `ORIG`. This bug is related to choosing a wrong
  instruction template. Running the PLV  pipeline will result in matcher bug,
  as follows:
```
cd  /home/sdasgup3/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark
../../scripts/run_batch_injected_bug_plv.sh docs/AE_docs/sampleInjectedBugs.txt
```

However, commenting-in the line below the `INJECTED_BUG` tag, and uncommenting
the line below `ORIG` tag, will make the matcher pass in all the cases.

For example, follow the steps above steps to rectify `~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Inject.1/binary/test.mcsema.ll` and invoke:
```
echo Inject.1/Rand/ > /dev/stdout | ../../scripts/run_batch_injected_bug_plv.sh  /dev/stdin
```

**Note**
The script `../../scripts/run_batch_injected_bug_plv.sh` differs from
`../../scripts/run_batch_injected_bug_plv.sh` in having an extra Make target
`mcsema`. From the [above](https://github.com/sdasgup3/PLDI20-Artifact-Evaluation/blob/master/README.md#testing-arena-for-plv) discussion, this target is used to (1) Invoke IDA +
McSema to generate `binary/test.mcsema.ll`, and (2) Sanitize the McSema generate
file to `binary/test.mcsema.inline.ll`. For reasons about IDA lisensing, we want
to skip (1). However, (2) is required, as we want to sanitize the bug-injected
McSema lifted file `binary/test.mcsema.ll`. To accomplish this partial
execution of Make target, we modified the corresponding [Makefile](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/program_translation_validation/single-source-benchmark/Inject.1/Makefile#L31);

## Single-Instruction validation (SIV)
### Source code
An important component of the SIV is the tool [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp), which converts the symbolic summary (specified in K-AST) to SMTLIB queries, with the core functionality defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/smt-generator/smt-generator.cpp).

### Testing Arena for SIV
For each binary instruction, for example the [addq_r64_r64](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/register-variants/adcq_r64_r64), has the following structure:

 - `test.c`: C file with an instruction instance of `addq %rcx, %rbx` wrapped in inline assembly.
 - `test`: Binary compiled from above C file.
 - `test.ll`: McSema lifted LLVM IR from binary `test`.
 - `test.mod.ll`: McSema lifts a lot af glue code other than the instructions in `test`. This is de-cluttered version of `test.ll` focusing on just the IR relevant to the lifting of `addq` instruction.
 - `test-xspec.k`: The K-specification file necessary to run a symbolic execution engine `krpove` (a tool that K-framework generates automatically from the semantics of X86-64 ISA) on binary instruction and to generate symbolic summary.
 - `Output/test-xspec.out`: Output of the above symbolic execution.
 - `test-lspec.k`: The K-specification file necessary to run another symbolic execution engine `kprove` (a tool that K-framework generates automatically from the semantics of LLVM IR) on `test.mod.ll` and to generate symbolic summary.
 - `Output/test-lspec.out`: Output of the above symbolic execution.
 - `Output/test-z3.py`: The python file encoding the **verification queries** in SMTLIB format. The queries are created using the tool [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp) by processing the symbolic summaries in  `Output/test-xspec.out` and `Output/test-lspec.out`.
 - `Makefile`: With the following relevant targets
   - `binary`: Generate `test` from `test.c`.
   - `mcsema`: Lift `test` to `test.ll` using McSema. (To Be Skipped)
   - `declutter`: Sanitize `test.ll` to `test.mod.ll`.
   - `genxspec`: Genearte the `test-xspec.k` file.
   - `collect`: Prepare for building the symbolic execution engine using the semantics definitions of the binary instructions included in `test`.
   - `kompile`: Generate the symbolic execution engine for binary instructions included in `test`. 
   - `kli`: Run concrete execution of `test.mod.ll` using the LLVM semantics. Output is stored in `Output/test-lstate.out`
   - `genlspec`: Generate the `test-lspec.k` file using many details from the concrete execution log `Output/test-lstate.out`.
   - `lprove`: Invoke symbolic execution using the spec file `test-lspec.k`. Generates `Output/test-lspec.out` containing the symbolic summary for the LLVM ir `test.mod.ll`.
   - `xprove`: Invoke symbolic execution using the spec file `test-xspec.k`. Generates `Output/test-xspec.out` containing the symbolic summary for the binary instruction `addq`.
   - `genz3`: Invoke [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp) to create `Output/test-z3.py` containing verification queries.
   - `provez3`: Invoke z3 on `Output/test-z3.py`.
   
**Note** 
  1. Similar files are available for other instructions and their variants (memory, immediate, register).
  2. We have pre-populated the McSema lifted LLVM IR `<instruction opcode>/test.ll` because McSema is not included in the VM distribution.
 

### Running the SIV pipeline

#### An example run
Running SIV on an isolated example instruction `addq_r64_r64` involves the following step
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
echo register-variants/addq_r64_r64 > /tmp/sample.txt
# Or try one of the following
# echo immediate-variants/addq_r64_imm32 > /tmp/sample.txt
# echo memory-variants/addq_r64_m64 > /tmp/sample.txt
../../scripts/run_batch_siv.sh /tmp/sample.txt |& tee ~/Junk/log

# cat ../../scripts/run_batch_siv.sh
## LIST=$1
## P=$2
## 
## if [ -z "$P" ]; then
##   P=1
## fi
## 
## TESTARENA="~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/" 
## 
## echo
## echo "Cleaning Stale instr semantics definitions"
## cd ~/Github/X86-64-semantics/semantics
## rm -rf underTestInstructions/*
## cd -
## 
## echo
## echo "Collecting instructions from binary"
## cat $LIST | parallel "cd $TESTARENA/{}; make collect; cd -"
## 
## echo
## echo "Kompiling the collected X86 semantics to create a sym-ex"
## cd ~/Github/X86-64-semantics/semantics
## ../scripts/kompile.pl --backend java
## cd -
## 
## echo
## echo "Batch Run Begin using $P jobs in parallel"
## 
## cat $LIST | parallel -j $P "echo ; echo {}; echo ======; cd ${TESTARENA}/{}; \
##       echo; echo \"Generating symbolic summary for binary instruction\" \
##       make genxspec; make xprove; \
##       echo; echo \"Generating symbolic summary for lifted LLVM IR\" \
##       make declutter; make kli; make genlspec; make lprove; \
##       echo; echo \"Generating verification conditions\" \
##       make genz3; \
##       echo; echo \"Prove verification conditions\" \
##       make provez3; \
##       cd -"
## 
## echo "Batch Run End"
```

**Note**
The Make target `xprove` will exit with non-zero status and it is a know issue.
These errors are because of missing KAST (internal K AST) to SMT
translation.  We do not need the translations as we have separate tool
[spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp)
to achieve the translation from the summary that the target `xprove` produces.
However, please note that all these error messages can be safely ignored as it
does NOT affect the soundness of the summary generation process.

#### Batch runs
The goal of the entire SIV pipeline is to genarate the file `Output/test-z3.py`
[an
example](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/single_instruction_translation_validation/mcsema/register-variants/adcq_r64_r64/Output/test-z3.py)
encoding the verification conditions, which is then dispatched (using Make
target `provez3`) to z3 for verification.

The python file has the following format:

```python 
 s = z3.solver()
 For each E in {registers, flags, memory values}: s.push()
  s.push()
  lvar = symbolic summary corresponding to E, obtained by sym-execution of the LLVM IR (generated by McSema by lifting the binary instruction)
  xvar = symbolic summary corresponding to E, obtained by sym-execution the binary instruction.

  s.add(lvar != xvar)

  solve s for unsat/sat/timeout

  s.pop()
```
Upon dispatch of the verification queries, for each register/flag/memory
values,  the output `Test-Pass` is generated if all the queries results in
`unsat`. Conversely, the output says `Test-Fail` if any of query results in
`sat or timeout`.

In order to run this prover step on a random sample of [non-buggy
instructions](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/non-bugs.txt),
do the following:
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
sort -R docs/AE_docs/non-bugs.txt | head -n 50 | parallel "echo ; echo {}; echo ===; cd {}; make provez3; cd -" |& tee ~/Junk/log
```

In order to run the entire SIV pileline (and hence to reproduce the
    verification conditions file)
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
sort -R docs/AE_docs/non-bugs.txt | head -n 4 > /tmp/sample.txt    # Select 4 random cases; Change it to any numer of random selection
../../scripts/run_batch_siv.sh /tmp/sample.txt 1 |& tee ~/Junk/log # If the guest machine is configured with more CPUs and RAM (>8GB), then the
                                                                   # reviewer is encouraged to fire more paralle runs (by changing 1 to a higher number)
```

#### Reproducing bugs
We provide the [a list of bugs](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/bugs.txt). In order to show
that the result of Z3-comparison on the corresponding verification queries, do the following:
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
cat docs/AE_docs/bugs.txt | parallel "echo ; echo {}; echo ===; cd {}; make provez3; cd -" |& tee ~/Junk/log
```
The above command will show, for each buggy instruction, the register has
mismatching symbolic summaries.  We know, from our past experience, that having
both the operands equal makes the semantics of some instructions pretty
challenging. We leverage that experience by running SIV on instructions with
its operands being the same register and found couple of bugs mentioned in the
above list as `register-variants-samereg/*`.  Bug Reports to McSema can be
                                          found
                                          [here](https://github.com/lifting-bits/remill/issues/376),
                                          which are all acknowledged.

#### Reproducing timeouts
The timeouts (related to mulq) include solver constraints containing bit vector
multiplication which the state-of-the-art SMT solvers are not very efficient at
reasoning about. These cases timed-out (after 24h) reproducibly.
```
cd register-variants/mulq_r64
make provez3 // Timeout provides is 24 hrs
```
