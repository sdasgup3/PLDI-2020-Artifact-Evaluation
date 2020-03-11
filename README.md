This file gives the step-by-step instructions on how  to evaluate the artifacts
for "Scalable Validation of Binary Lifters".

## Artifact Submission
 - Accepted Paper [pdf](https://github.com/sdasgup3/PLDI-2020-Artifact-Evaluation/blob/master/pldi2020-paper29.pdf)
 - VM Details
    - VM Player: [VirtualBox](https://www.virtualbox.org/) 6.1 Or 5.1
    - Ubuntu Image: [ova, size ~16G ](https://drive.google.com/file/d/137gurDl7OvfxS1fEq9_qCJLt3RdsrZaS/view?usp=sharing)
      -   [md5 hash](https://docs.google.com/document/d/1YzOBUxWMoXes9bcqWTJYrXeWxVRGFUnefhs8jDxDabs/edit?usp=sharing)
      -   login: sdasgup3
      -   password: aecadmin123
    - Guest Machine recommended settings
      - RAM: Min-8GB, Max-16GB and Cores:Min-4, Max-8. However, it is recommended to configure the VM with more RAM and processors to allow  parallel experiments during single instruction validation.
      - Allow 70GB of disk space as needed by the unpacked VM.
  - We have also included the current repository in the VM disk, so that one can access this README.md file at `~/Github/PLDI20-Artifact-Evaluation/README.md`. This is just only to avoid the unfortunate scenario of the "bidirectional shared clipboard" not working for the VirtualBox.
  
**Troubleshoot**: 
1. For a Ubuntu host machine with Secure Boot enabled, the presented VirtualBox image may fail to be loaded. In that case, you can either disable the Secure Boot, or sign the VirtualBox module as described [here](https://askubuntu.com/questions/900118/vboxdrv-sh-failed-modprobe-vboxdrv-failed-please-use-dmesg-to-find-out-why/900121#900121).
2. [VM boot up with black login screen](https://askubuntu.com/questions/1134892/ubuntu-18-04-lts-on-virtualbox-boots-up-but-black-login-screen)
3. Virtualbox `Error ID: BLKCACHE_IOERR`: In the VB client, Storage » SATA Controller" Use the cache I/O host (all other values are those used by default VirtualBox))
4. On startup of the VM, you will get the following warning, which we recommend to ignore.
```
On startup of the VM, I received this error message (although otherwise the VM functioned normally):
Error found when loading /home/sdasgup3/.profile:

tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
/home/sdasgup3/.bashrc: line 73: bind: warning: line editing not enabled
Could not open a connection to your authentication agent.
Could not open a connection to your authentication agent.

As a result the session will not be configured correctly.
You should fix the problem as soon as feasible
```

# Getting Started
To check if the installations are working as expected, the reviewers should be able to run the following steps. 
The steps (detailed in [Step-by-step Instructions](https://github.com/sdasgup3/PLDI20-Artifact-Evaluation/blob/master/README.md#step-by-step-instructions) section) are responsible to run the program-level validation
on function `Queen::Doit`
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/
make compd
make match
```
and expect the following output
```
$ export NORM=CUSTOM

$ cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/

$ make compd
mkdir -p mcsema/
time /home/sdasgup3/Github//validating-binary-decompilation/source/build/bin//decompiler  --output mcsema/test.proposed.ll --path /home/sdasgup3/Github/compd_cache/ --function Doit --input ../binary/test.reloc --use-reloc-info 1>mcsema/compd.log 2>&1
Compd Pass:- /home/sdasgup3/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit:Doit
opt -S  -inline   mcsema/test.proposed.ll -o mcsema/test.proposed.inline.ll

$ make match
opt -S -mem2reg -licm -gvn -early-cse -globalopt -simplifycfg -basicaa -aa -memdep -dse -deadargelim -libcalls-shrinkwrap -tailcallelim -simplifycfg -basicaa -aa -instcombine -simplifycfg -early-cse -gvn -basicaa -aa -memdep -dse -memcpyopt mcsema/test.proposed.inline.ll -o mcsema/test.proposed.opt.ll
opt -S -mem2reg -licm -gvn -early-cse -globalopt -simplifycfg -basicaa -aa -memdep -dse -deadargelim -libcalls-shrinkwrap -tailcallelim -simplifycfg -basicaa -aa -instcombine -simplifycfg -early-cse -gvn -basicaa -aa -memdep -dse -memcpyopt ../binary/test.mcsema.inline.ll -o mcsema/test.mcsema.opt.ll
( time /home/sdasgup3/Github//validating-binary-decompilation/source/build/bin//matcher --file1 mcsema/test.mcsema.opt.ll:Doit --file2 mcsema/test.proposed.opt.ll:Doit ) 1>mcsema/match_mcsema_proposed.log 2>&1
( time /home/sdasgup3/Github//validating-binary-decompilation/source/build/bin//matcher --file1 mcsema/test.proposed.opt.ll:Doit --file2 mcsema/test.mcsema.opt.ll:Doit ) 1>mcsema/match_proposed_mcsema.log 2>&1
Match Pass:both-exact-match:- /home/sdasgup3/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit:Doit
```

# Claims
The reviewer should be able to 
1. reproduce the program-level validation (PLV) runs on single-source benchmark.
2. check that PLV is effective in detecting bugs which are artificially injected.
3. run single-instruction validation pipeline on individual instructions and generate the verification conditions to be dispatched to z3 solver.
4. reproduce bugs and timeouts reported in the paper.

# Step-by-step Instructions
## Program-Level validation (PLV)
### Source code

The two major components of PLV are [Compositional Lifter](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/decompiler/decompiler.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/compositional-decompiler/compositional-decompiler.cpp) and [matcher](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/matcher/matcher.cpp), with the core logic defined in [library](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/libs/llvm-graph-matching/llvm-graph-matching.cpp). Additionally, Compositional Lifter uses a [_Store_](https://github.com/sdasgup3/compd_cache) of formally validated instructions to stitch the validated LLVM IR sequence of constituent instructions in a binary program.

### Testing arena for PLV

The [test directory](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/) contains test-suites like `toy-examples` and `single-source-benchmark`. The [single-source-benchmark](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark) contain folders for all the programs hosted by the test-suite. Each such program, for example the [Queens program](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/program_translation_validation/single-source-benchmark/Queens), has the following structure:


 - `src/` (the source artifacts of the program)
   - `test.ll or test.bc` (Source LLVM code of `Queens`'s program) 
 - `binary/`
   - `test`: Binary compiled from src/test.bc
   - `test.mcsema.bc`: McSema lifted LLVM IR from `binary/test`
   - `test.mcsema.calls_renamed.ll`: A version of `binary/test.mcsema.ll` with function calls renamed so as to prevent inter-procedural optimization during normalization
   - `test.mcsema.inline.ll`: Inlined version of `binary/test.mcsema.calls_renamed.ll`
   - `test.reloc`: Binary compiled from src/test.bc with relocation information preserved
 - `Makefile`: With the following relevant targets
   - `binary`: To generate `binary/test`
   - `reloc_binary`: To generate `binary/test.reloc`
   - `mcsema`: To generate `binary/test.mcsema.bc`,`binary/test.mcsema.inline.ll`, and `binary/test.mcsema.calls_renamed.ll` using McSema tool (To be Skipped during evaluation for reason detailed below)
  - `Doit/ (Similarly for Initrand/, Rand/, main/, Try/, or Queens/)` (Artifacts related to individual functions of the `Queens`'s program, extracted using external tool [wllvm](https://github.com/travitch/whole-program-llvm))
    - `mcsema/test.proposed.ll`: The LLVM IR, corresponding to function `Doit`, generated using Compositional Lifter.
    - `mcsema/test.proposed.inlined.ll`: Inlined version of `mcsema/test.proposed.ll`
    - `Makefile`: With the following relevant targets
      - `compd`: Invoke Compositional Lifter to generate `mcsema/test.proposed.inline.ll`
      - `compd_opt`: Invoke Normalizer to normalize `mcsema/test.proposed.inline.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/pldi20_ae/tests/scripts/matcher_driver.sh#L16). Generates `mcsema/test.proposed.opt.ll`.
      - `mcsema_opt`: Invoke Normalizer to normalize `../binary/test.mcsema.inline.ll` using the [LLVM opt pass list](https://github.com/sdasgup3/validating-binary-decompilation/blob/pldi20_ae/tests/scripts/matcher_driver.sh#L16). Generates `mcsema/test.mcsema.opt.ll`
      - `match`: Invoke Matcher to check for graph-isomorphism on the data-dependence graphs of the above normalized versions `mcsema/test.proposed.opt.ll` & `mcsema/test.mcsema.opt.ll`.

**Note** 
 1. We have pre-populated the McSema lifted LLVM IR `<program name>/binary/test.mcsema.ll` because McSema needs a licensed disassembler `IDA` to generate this file, which is not provided because of some licensing issues. Hence, the Make target `mcsema` will not work.
 2. The [_Store_](https://github.com/sdasgup3/compd_cache) has the validated IR sequences of individual binary instructions,  generated using McSema. For similar reasons as above, we have packaged the entire _Store_ in the VM, so that the reviewer do not have to invoke McSema. 
 
### Running the PLV pipeline

#### An example run
Here we will elaborate the process of running PLV on an isolated example function `Queens/Doit/`. 
We use shell variable NORM to specify which set of optimization passes to use for normalization. For example, the value `CUSTOM` enables using a [set of 17 LLVM opt passes](https://github.com/sdasgup3/validating-binary-decompilation/blob/pldi20_ae/tests/scripts/matcher_driver.sh#L16) for normalization. As as aside, there is an option for NORM which enables AutoTuner for pass selection.

Running PLV on it involves the following steps
```
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Queens/Doit/

## Running Mcsema to lift the binary "../binary/test"
## Creates  "../binary/test.mcsema.inline.ll" (already populated)
# make mcsema // Skip  this step as mentioned above

## Running Compositional lifter on the corresponding binary ../binary/test.reloc
## Creates mcsema/test.proposed.inline.ll
make compd

## Normalizing mcsema/test.proposed.inline.ll
## Creates mcsema/test.proposed.opt.ll
make compd_opt # expect "Compd Pass" upon execution

## Normalizing ../binary/test.mcsema.inline.ll (Already populated using make mcsema)
## Creates mcsema/test.mcsema.opt.ll
make mcsema_opt

## Matching mcsema/test.proposed.opt.ll & mcsema/test.mcsema.opt.ll
make match # expect "Match Pass" upon execution
```
**Note**
The Make target `match` already includes the normalization actions of
`compd_opt` & `mcsema_opt`, hence one can skip invoking the redundant targets.
The reason we still have those targets are for legacy reasons and included in
this presentation for better explanation of the steps. For example, instead of
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
      using the Compositional Lifter, is `32105` LOC. Note that, the runtime of the matcher on 
      `himenobmtxpa/jacobi` might take up-to 2 mins.


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
export NORM=CUSTOM
cd ~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/
sort -R docs/AE_docs/matcherPassList.txt | head -n 10 > /dev/stdout | ../../scripts/run_batch_plv.sh   /dev/stdin |& tee ~/Junk/log
```

### PLV injected bug detection
We provided a [list of
test-cases](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/program_translation_validation/single-source-benchmark/docs/AE_docs/sampleInjectedBugs.txt),
to demonstrate the effectiveness of PLV to catch potential bugs. The test-cases
represent the four different category of injections as mentioned in paper (line
    942) and generated by modifying the McSema-lifted-IR function
`Queens/Rand`.

For example, an example injection can be found at
`~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Inject.1/binary/test.mcsema.ll`,
  where the bug is marked with comment tag `INJECTED_BUG`, along with the original line
  of code marked by tag `ORIG`. This bug is related to choosing a wrong
  instruction template. Running the PLV  pipeline on this (and other injected bugs) will result in matcher bug,
  as follows:
```
cd  /home/sdasgup3/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark
../../scripts/run_batch_injected_bug_plv.sh docs/AE_docs/sampleInjectedBugs.txt |& tee ~/Junk/log
```

However, commenting-in the line below the `INJECTED_BUG` tag, and un-commenting
the line below `ORIG` tag, will make the matcher pass in all the cases.

For example, follow the steps above steps to rectify `~/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark/Inject.1/binary/test.mcsema.ll` and invoke:
```
cd  /home/sdasgup3/Github/validating-binary-decompilation/tests/program_translation_validation/single-source-benchmark
echo Inject.1/Rand/ > /dev/stdout | ../../scripts/run_batch_injected_bug_plv.sh  /dev/stdin
```

**Note**
The script `../../scripts/run_batch_plv.sh` differs from
`../../scripts/run_batch_injected_bug_plv.sh` in having an extra Make target
`mcsema`. From the [above](https://github.com/sdasgup3/PLDI20-Artifact-Evaluation/blob/master/README.md#testing-arena-for-plv) discussion, this target is used to (1) Invoke IDA +
McSema to generate `binary/test.mcsema.ll`, and (2) Sanitize the McSema generate
file to `binary/test.mcsema.inline.ll`. For reasons about IDA licensing, we want
to skip (1). However, (2) is required, as we want to sanitize the modified
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
   - Target related to generating symbolic summary for binary instruction
      - `collect`: Prepare for building the symbolic execution engine using the semantics definitions of the binary instructions included in `test`.
      - `kompile`: Generate the symbolic execution engine for binary instructions included in `test`.
      - `genxspec`: Generate the `test-xspec.k` file.
      - `xprove`: Invoke symbolic execution using the spec file `test-xspec.k`. Generates `Output/test-xspec.out` containing the symbolic summary for the binary instruction `addq`.
   - Target related to generating symbolic summary for lifted LLVM IR
      - `mcsema`: Lift `test` to `test.ll` using McSema. (To Be Skipped because of IDA licensing)
      - `declutter`: Sanitize `test.ll` to `test.mod.ll`.
      - `kli`: Run concrete execution of `test.mod.ll` using the LLVM semantics. Output is stored in `Output/test-lstate.out`
      - `genlspec`: Generate the `test-lspec.k` file using many details from the concrete execution log `Output/test-lstate.out`.
      - `lprove`: Invoke symbolic execution using the spec file `test-lspec.k`. Generates `Output/test-lspec.out` containing the symbolic summary for the lifted LLVM IR `test.mod.ll`.
   - Target related to generating and proving verification conditions
      - `genz3`: Invoke [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp) to create `Output/test-z3.py` containing verification queries.
      - `provez3`: Invoke z3 on `Output/test-z3.py`.

**Note**
  1. Similar files are available for other instructions and their variants (memory/immediate/register).
  2. We have pre-populated the McSema lifted LLVM IR `<instruction opcode>/test.ll` because McSema is not included in the VM distribution.

### Running the SIV pipeline

#### An example run
Running SIV on an isolated example instruction `addq_r64_r64` involves the following step
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
echo register-variants/addq_r64_r64 > /tmp/sample.txt
# Or try one of the following
#     echo immediate-variants/addq_r64_imm32 > /tmp/sample.txt        # Try an immediate variant
#     echo memory-variants/addq_r64_m64 > /tmp/sample.txt             # Try a memory variant
#     sort -R docs/AE_docs/non-bugs.txt | head -n 1 > /tmp/sample.txt # Try any random instruction from a list
                                                                      # of passing cases

../../scripts/run_batch_siv.sh /tmp/sample.txt |& tee ~/Junk/log

# cat ../../scripts/run_batch_siv.sh
## LIST=$1
##
## ## Number of jobs to issue in parallel
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
## echo "Compiling the collected X86 semantics to create a sym-ex"
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
1. The Make target `xprove` will exit with non-zero status and it is a known issue.
  These errors are because of missing KAST (internal K AST) to SMT
  translations in the sym-ex engine backend. We do not need the translations as we have a separate tool
  [spec-to-smt](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/source/tools/spec-to-smt/spec-to-smt.cpp)
  to achieve the translation from the summary that the target `xprove` produces.
  However, please note that all these error messages can be safely ignored as it
  does NOT affect the soundness of the summary generation process.

2. The entire execution will take up-to 4-5 mins.

#### Batch runs
The goal of the entire SIV pipeline (as mentioned above) is to reproduce the file verification-query file `Output/test-z3.py`
from scratch.
Here is [an
example test-z3.py file](https://github.com/sdasgup3/validating-binary-decompilation/blob/master/tests/single_instruction_translation_validation/mcsema/register-variants/adcq_r64_r64/Output/test-z3.py)
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

#### Reproducing bugs
We provide the [a list of bugs](https://github.com/sdasgup3/validating-binary-decompilation/tree/master/tests/single_instruction_translation_validation/mcsema/docs/AE_docs/bugs.txt). In order to demonstrate
 the result of Z3-comparison on the corresponding verification queries, do the following:
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/
cat docs/AE_docs/bugs.txt | parallel "echo ; echo {}; echo ===; cd {}; make provez3; cd -" |& tee ~/Junk/log
```
The above command will show, for each buggy instruction, the register that has a
mismatching symbolic summary.  We know, from our past experience, that having
both the operands equal makes the semantics of some instructions pretty
challenging. We leverage that experience by running SIV on instructions with
its operands being the same registers and found couple of bugs mentioned in the
above list as `register-variants-samereg/*`.  Bug Reports to McSema can be
                                          found
                                          [here](https://github.com/lifting-bits/remill/issues/376),
                                          which are all acknowledged.

#### Reproducing timeouts
The timeouts (related to `mulq`) include solver constraints containing bit vector
multiplication which the state-of-the-art SMT solvers are not very efficient at
reasoning about. These cases timed-out (after 24h) reproducibly.
```
cd ~/Github/validating-binary-decompilation/tests/single_instruction_translation_validation/mcsema/register-variants/mulq_r64
make provez3 ## Timeout provides is 24 hrs
```
