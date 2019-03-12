# Writing a Pass

[TOC]

Passes represent the basic infrastructure for transformation and optimization.
This document provides a quickstart to the pass infrastructure in MLIR and how
to use it.

See [MLIR specification](LangRef.md) for more information about MLIR and its
core aspects, such as the IR structure and operations.

See [MLIR Rewrites](QuickstartRewrites.md) for a quick start on graph rewriting
in MLIR. If your transformation involves pattern matching operation DAGs, this
is a great place to start.

## Pass Types {#pass-types}

MLIR provides different pass classes for several different granularities of
transformation. Depending on the granularity of the transformation being
performed, a pass may derive from [FunctionPass](#function-pass) or
[ModulePass](#module-pass); with each requiring a different set of constraints.

### FunctionPass {#function-pass}

A function pass operates on a per-function granularity, executing on
non-external functions within a module in no particular order. Function passes
have the following restrictions, and any noncompliance will lead to problematic
behavior in multithreaded and other advanced scenarios:

*   Modify anything within the parent module, outside of the current function
    being operated on. This includes adding or removing functions from the
    module.
*   Maintain pass state across invocations of runOnFunction. A pass may be run
    on several different functions with no guarantee of execution order.
    *   When multithreading, a specific pass instance may not even execute on
        all functions within the module. As such, a function pass should not
        rely on running on all functions within a module.
*   Access, or modify, the state of another function within the module.
    *   Other threads may be operating on different functions within the module.
*   Maintain any global mutable state, e.g. static variables within the source
    file. All mutable state should be maintained by an instance of the pass.

To create a function pass, a derived class must adhere to the following:

*   Inherit from the CRTP class `FunctionPass`.
*   Override the virtual `void runOnFunction()` method.
*   Must be copy-constructible, multiple instances of the pass may be created by
    the pass manager to process functions in parallel.

A simple function pass may look like:

```c++
namespace {
struct MyFunctionPass : public FunctionPass<MyFunctionPass> {
  void runOnFunction() override {
    // Get the current function being operated on.
    Function *f = getFunction();

    // Operate on the instructions within the function.
    f->walk([](Instruction *inst) {
      ....
    });
  }
};
} // end anonymous namespace

// Register this pass to make it accessible to utilities like mlir-opt.
// (Pass registration is discussed more below)
static PassRegistration<MyFunctionPass> pass(
    "flag-name-to-invoke-pass-via-mlir-opt", "Pass description here");
```

### ModulePass {#module-pass}

A module pass operates on a per-module granularity, executing on the entire
program as a unit. As such, module passes are able to add/remove/modify any
functions freely. Module passes have the following restrictions, and any
noncompliance will lead to problematic behavior in multithreaded and other
advanced scenarios:

*   Maintain pass state across invocations of runOnModule.
*   Maintain any global mutable state, e.g. static variables within the source
    file. All mutable state should be maintained by an instance of the pass.

To create a module pass, a derived class must adhere to the following:

*   Inherit from the CRTP class `ModulePass`.
*   Override the virtual `void runOnModule()` method.

A simple module pass may look like:

```c++
namespace {
struct MyModulePass : public ModulePass<MyModulePass> {
  void runOnModule() override {
    // Get the current module being operated on.
    Module *m = getModule();

    // Operate on the functions within the module.
    for (auto &func : *m) {
      ....
    }
  }
};
} // end anonymous namespace

// Register this pass to make it accessible to utilities like mlir-opt.
// (Pass registration is discussed more below)
static PassRegistration<MyModulePass> pass(
    "flag-name-to-invoke-pass-via-mlir-opt", "Pass description here");
```

## Analysis Management

An important concept, along with transformation passes, are analyses. These are
conceptually similar to transformation passes, except that they compute
information on a specific Function, or Module, without modifying it. In MLIR,
analyses are not passes but free standing classes that are computed lazily
on-demand and cached to avoid unnecessary recomputation. An analysis in MLIR
must adhere to the following:

*   Provide a valid constructor for a given IR unit.
*   Must not modify the given IR unit.

Each of the base Pass classes provide utilities for querying and preserving
analyses for the current IR being processed. Using the example passes defined
above, let's see some examples:

### Querying Analyses

*   FunctionPass automatically provides the following utilities for querying
    analyses:
    *   `getAnalysis<>`
    *   `getCachedAnalysis<>`
    *   `getCachedModuleAnalysis<>`
*   ModulePass automatically provides the following utilities:
    *   `getAnalysis<>`
    *   `getCachedAnalysis<>`
    *   `getFunctionAnalysis<>`
    *   `getCachedFunctionAnalysis<>`

```c++
/// An interesting function analysis.
struct MyFunctionAnalysis {
  // Compute this analysis with the provided function.
  MyFunctionAnalysis(Function *function);
};

/// An interesting module analysis.
struct MyModuleAnalysis {
  // Compute this analysis with the provided module.
  MyModuleAnalysis(Module *module);
};

void MyFunctionPass::runOnFunction() {
  // Query MyFunctionAnalysis for the current function.
  MyFunctionAnalysis &myAnalysis = getAnalysis<MyFunctionAnalysis>();

  // Query a cached instance of MyFunctionAnalysis for the current function. It
  // will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedAnalysis<MyFunctionAnalysis>();
  if (optionalAnalysis)
    ...

  // Query a cached instance of MyModuleAnalysis for the parent module of the
  // current function. It will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedModuleAnalysis<MyModuleAnalysis>();
  if (optionalAnalysis)
    ...
}

void MyModulePass::runOnModule() {
  // Query MyModuleAnalysis for the current module.
  MyModuleAnalysis &myAnalysis = getAnalysis<MyModuleAnalysis>();

  // Query a cached instance of MyModuleAnalysis for the current module. It
  // will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedAnalysis<MyModuleAnalysis>();
  if (optionalAnalysis)
    ...

  // Query MyFunctionAnalysis for a child function of the current module. It
  // will be computed if it doesn't exist.
  auto *fn = &*getModule().begin();
  MyFunctionAnalysis &myAnalysis = getFunctionAnalysis<MyFunctionAnalysis>(fn);
}
```

### Preserving Analyses

Analyses that are constructed after being queried by a pass are cached to avoid
unnecessary computation if they are requested again later. To avoid stale
analyses, all analyses are assumed to be invalidated by a pass. To avoid
invalidation, a pass must specifically mark analyses that are known to be
preserved.

*   All Pass classes automatically provide the following utilities for
    preserving analyses:
    *   `markAllAnalysesPreserved`
    *   `markAnalysesPreserved<>`

```c++
void MyPass::runOn*() {
  // Mark all analyses as preserved. This is useful if a pass can guarantee
  // that no transformation was performed.
  markAllAnalysesPreserved();

  // Mark specific analyses as preserved. This is used if some transformation
  // was performed, but some analyses were either unaffected or explicitly
  // preserved.
  markAnalysesPreserved<MyAnalysis, MyAnalyses...>();
}
```

## Pass Failure

Passes in MLIR are allowed to gracefully fail. This may happen if some invariant
of the pass was broken, potentially leaving the IR in some invalid state. If
such a situation occurs, the pass can directly signal a failure to the pass
manager. If a pass signaled a failure when executing, no other passes in the
pipeline will execute and the `PassManager::run` will return failure. Failure
signaling is provided in the form of a `signalPassFailure` method.

```c++
void MyPass::runOn*() {
  // Signal failure on a broken invariant.
  if (some_broken_invariant) {
    signalPassFailure();
    return;
  }
}
```

## Pass Manager {#pass-manager}

Above we introduced the different types of passes and their constraints. Now
that we have our pass we need to be able to run it over a specific module. This
is where the pass manager comes into play. The PassManager is the interface used
for scheduling passes to run over a module. The pass manager provides simple
interfaces for adding passes, of any kind, and running them over a given module.
A simple example is shown below:

```c++
PassManager pm;

// Add a module pass.
pm.addPass(new MyModulePass());

// Add a few function passes.
pm.addPass(new MyFunctionPass());
pm.addPass(new MyFunctionPass2());
pm.addPass(new MyFunctionPass3());

// Add another module pass.
pm.addPass(new MyModulePass2());

// Run the pass manager on a module.
Module *m = ...;
if (failed(pm.run(m)))
    ... // One of the passes signaled a failure.
```

The pass manager automatically structures added passes into nested pipelines on
specific IR units. These pipelines are then run over a single IR unit at a time.
This means that, for example, given a series of consecutive function passes, it
will execute all on the first function, then all on the second function, etc.
until the entire program has been run through the passes. This provides several
benefits:

*   This improves the cache behavior of the compiler, because it is only
    touching a single function at a time, instead of traversing the entire
    program.
*   This improves multi-threading performance by reducing the number of jobs
    that need to be scheduled, as well as increasing the efficency of each job.
    An entire function pipeline can be run on each function asynchronously.

As an example, the above pass manager would contain the following pipeline
structure:

```c++
MyModulePass
Function Pipeline
   MyFunctionPass
   MyFunctionPass2
   MyFunctionPass3
MyModulePass2
```

## Pass Registration {pass-registration}

Briefly shown in the example definitions of the various
[pass types](#pass-types) is the `PassRegistration` class. This is a utility to
register derived pass classes so that they may be created, and inspected, by
utilities like mlir-opt. Registering a pass class takes the form:

```c++
static PassRegistration<MyPass> pass("command-line-arg", "description");
```

*   `MyPass` is the name of the derived pass class.
*   "command-line-arg" is the argument to use on the command line to invoke the
    pass from `mlir-opt`.
*   "description" is a description of the pass.

### Pass Pipeline Registration

Described above is the mechanism used for registering a specific derived pass
class. On top of that, MLIR allows for registering custom pass pipelines in a
similar fashion. This allows for custom pipelines to be available to tools like
mlir-opt in the same way that passes are, which is useful for encapsulating
common pipelines like the "-O1" series of passes. Pipelines are registered via a
similar mechanism to passes in the form of `PassPipelineRegistration`. Compared
to `PassRegistration`, this class takes an additional parameter in the form of a
pipeline builder that modifies a provided PassManager.

```c++
void pipelineBuilder(PassManager &pm) {
  pm.addPass(new MyPass());
  pm.addPass(new MyOtherPass());
}

// Register an existing pipeline builder function.
static PassPipelineRegistration pipeline(
  "command-line-arg", "description", pipelineBuilder);

// Register an inline pipeline builder.
static PassPipelineRegistration pipeline(
  "command-line-arg", "description", [](PassManager &pm) {
    pm.addPass(new MyPass());
    pm.addPass(new MyOtherPass());
  });
```

Pipeline registration also allows for simplified registration of
specifializations for existing passes:

```c++
static PassPipelineRegistration foo10(
    "foo-10", "Foo Pass 10", [] { return new FooPass(10); } );
```

## Pass Instrumentation

MLIR provides a customizable framework to instrument pass execution and analysis
computation. This is provided via the `PassInstrumentation` class. This class
provides hooks into the PassManager that observe various pass events:

*   `runBeforePass`
    *   This callback is run just before a pass is executed.
*   `runAfterPass`
    *   This callback is run right after a pass has been successfully executed.
        If this hook is executed, runAfterPassFailed will not be.
*   `runAfterPassFailed`
    *   This callback is run right after a pass execution fails. If this hook is
        executed, runAfterPass will not be.
*   `runBeforeAnalysis`
    *   This callback is run just before an analysis is computed.
*   `runAfterAnalysis`
    *   This callback is run right after an analysis is computed.

PassInstrumentation objects can be registered directly with a
[PassManager](#pass-manager) instance via the `addInstrumentation` method.
Instrumentations added to the PassManager are run in a stack like fashion, i.e.
the last instrumentation to execute a `runBefore*` hook will be the first to
execute the respective `runAfter*` hook. Below in an example instrumentation
that counts the number of times DominanceInfo is computed:

```c++
struct DominanceCounterInstrumentation : public PassInstrumentation {
  std::atomic<int> &count;

  DominanceCounterInstrumentation(std::atomic<int> &count) : count(count) {}
  void runAfterAnalysis(llvm::StringRef, AnalysisID *id,
                        const llvm::Any &) override {
    if (id == AnalysisID::getID<DominanceInfo>())
      ++count;
  }
};

PassManager pm;

// Add the instrumentation to the pass manager.
std::atomic<int> domInfoCount;
pm.addInstrumentation(new DominanceCounterInstrumentation(domInfoCount));

// Run the pass manager on a module.
Module *m = ...;
if (failed(pm.run(m)))
    ...

llvm::errs() << "DominanceInfo was computed " << domInfoCount << " times!\n";
```

### Standard Instrumentations

MLIR utilizes the pass instrumentation framework to provide a few useful
developer tools and utilites. Each of these instrumentations are immediately
available to all users of the MLIR pass framework.

#### Pass Timing

The PassTiming instrumentation provides timing information about the execution
of passes and computation of analyses. This provides a quick glimpse into what
passes are taking the most time to execute, as well as how much of an effect
your pass has on the total execution time of the pipeline. This instrumentation
is made available in mlir-opt via the `-pass-timing` and `-pass-timing-display`
flags. `-pass-timing-display` allows for toggling between the different display
modes for the timing results. The available display modes are described below:

##### List Display Mode

In this mode, the results are displayed in a list sorted by total time with each
pass/analysis instance aggregated into one unique result. This view is useful
for getting an overview of what analyses/passes are taking the most time in a
pipeline. This display mode is available in mlir-opt via
`-pass-timing-display=list`.

```shell
$ mlir-opt foo.mlir -cse -canonicalize -convert-to-llvmir -pass-timing -pass-timing-display=list

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0269 seconds (0.0269 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0102 ( 42.6%)   0.0017 ( 60.2%)   0.0119 ( 44.4%)   0.0120 ( 44.5%)  Canonicalizer
   0.0043 ( 17.9%)   0.0000 (  0.0%)   0.0043 ( 16.0%)   0.0043 ( 16.0%)  LLVMLowering
   0.0036 ( 14.9%)   0.0006 ( 22.8%)   0.0042 ( 15.8%)   0.0042 ( 15.7%)  FunctionVerifier
   0.0038 ( 15.6%)   0.0000 (  0.0%)   0.0038 ( 14.0%)   0.0038 ( 14.0%)  ModuleVerifier
   0.0013 (  5.5%)   0.0003 ( 10.4%)   0.0016 (  6.1%)   0.0016 (  6.1%)  CSE
   0.0008 (  3.4%)   0.0002 (  6.6%)   0.0010 (  3.7%)   0.0010 (  3.7%)  DominanceInfo
   0.0240 (100.0%)   0.0028 (100.0%)   0.0269 (100.0%)   0.0269 (100.0%)  Total
```

##### Pipeline Display Mode

In this mode, the results are displayed in a nested pipeline view that mirrors
the internal pass pipeline that is being executed in the pass manager. This view
is useful for understanding specifically which parts of the pipeline are taking
the most time, and can also be used to identify when analyses are being
invalidated and recomputed. This display mode is available in mlir-opt via
`-pass-timing-display=pipeline`.

```shell
$ mlir-opt foo.mlir -cse -canonicalize -convert-to-llvmir -pass-timing -pass-timing-display=pipeline

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0272 seconds (0.0272 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0119 ( 59.8%)   0.0066 ( 91.1%)   0.0185 ( 68.2%)   0.0185 ( 68.2%)  Function Pipeline
   0.0010 (  5.3%)   0.0006 (  8.3%)   0.0017 (  6.1%)   0.0016 (  6.1%)    CSE
   0.0006 (  3.2%)   0.0004 (  5.4%)   0.0010 (  3.8%)   0.0010 (  3.8%)      (A) DominanceInfo
   0.0013 (  6.4%)   0.0008 ( 10.4%)   0.0020 (  7.4%)   0.0020 (  7.4%)    FunctionVerifier
   0.0075 ( 37.9%)   0.0041 ( 55.9%)   0.0116 ( 42.7%)   0.0116 ( 42.8%)    Canonicalizer
   0.0014 (  7.0%)   0.0008 ( 11.2%)   0.0022 (  8.1%)   0.0022 (  8.1%)    FunctionVerifier
   0.0017 (  8.7%)   0.0000 (  0.0%)   0.0017 (  6.4%)   0.0017 (  6.4%)  ModuleVerifier
   0.0047 ( 23.5%)   0.0000 (  0.0%)   0.0047 ( 17.2%)   0.0047 ( 17.2%)  LLVMLowering
   0.0016 (  7.9%)   0.0006 (  8.9%)   0.0022 (  8.2%)   0.0022 (  8.2%)  ModuleVerifier
   0.0199 (100.0%)   0.0073 (100.0%)   0.0272 (100.0%)   0.0272 (100.0%)  Total
```