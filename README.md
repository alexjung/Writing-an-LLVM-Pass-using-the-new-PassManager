# Writing an LLVM Pass using the (kind of) new PassManager
This tutorial assumes you already have the LLVM project somewhere on your computer and that you are familiar with compiling it. If you don't, please refer to the [official documentation](http://releases.llvm.org/6.0.0/docs/GettingStarted.html) [1].

In general this tutorial is based on the official tutorial [Writing An LLVM Pass Tutorial](http://releases.llvm.org/6.0.0/docs/WritingAnLLVMPass.html) [2] from the LLVM documentation and the blog posts by Bekket McClane which can be found [here](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82) [3]. You should start with following those tutorials if you haven't already done that.  
But as the former lacks any information about the new PassManager and the latter does not cover integration of "new-PassManager-passes" into the LLVM source tree, I dug into the sources of LLVM and compiled all the information into the following tutorial.

## Writing a FunctionPass as a loadable shared library

*Note that this should also apply to ModulePass.*

This is the quickest way to write a pass and use it on LLVM bitcode (`.bc`) as it only requires one `.cpp` file and two small changes in the build environment of LLVM. Those changes can be done according to the official [Writing An LLVM Pass Tutorial](http://releases.llvm.org/6.0.0/docs/WritingAnLLVMPass.html) [2]:
- Create a directory `lib/Transforms/MyPass`

- Add a `CMakeLists.txt` file with the following content into the directory:
  ```cmake
  add_llvm_loadable_module( MyPass
      MyPass.cpp

      PLUGIN_TOOL
      opt
  )
  ```

- Add the following into `lib/Transforms/CMakeLists.txt`: `add_subdirectory(MyPass)`

- Add a source file `MyPass.cpp` in `lib/Transforms/MyPass` containing the following code:

  Adapted from the first part of Bekket McClane's [tutorial series](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82) [3]:

  ```cpp
  #include "llvm/IR/PassManager.h"
  #include "llvm/Passes/PassBuilder.h"
  #include "llvm/Passes/PassPlugin.h"
  #include "llvm/Support/raw_ostream.h"

  // only needed for printing
  #include  <iostream>

  using namespace llvm;

  namespace {

  struct MyPass : public PassInfoMixin<MyPass> {

    // The first argument of the run() function defines on what level
    // of granularity your pass will run (e.g. Module, Function).
    // The second argument is the corresponding AnalysisManager
    // (e.g ModuleAnalysisManager, FunctionAnalysisManager)
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM) {

      std::cout << "MyPass in function: " << F.getName().str() << std::endl;

      // Here goes what you want to do with a pass

      // Assuming you did not change anything of the IR code
      return PreservedAnalyses::all();
    }
  };
  }

  // This part is the new way of registering your pass
  extern "C" ::llvm::PassPluginLibraryInfo LLVM_ATTRIBUTE_WEAK
  llvmGetPassPluginInfo() {
    return {
      LLVM_PLUGIN_API_VERSION, "MyPass", "v0.1",
      [](PassBuilder &PB) {
        PB.registerPipelineParsingCallback(
          [](StringRef Name, FunctionPassManager &FPM,
          ArrayRef<PassBuilder::PipelineElement>) {
            if(Name == "my-pass"){
              FPM.addPass(MyPass());
              return true;
            }
            return false;
          }
        );
      }
    };
  }
  ```

Now you have written a simple FunctionPass and you can compile LLVM. The compilation process will generate a shared library `LLVMMyPass.so` in the `build/lib`directory. To run your pass on a bitcode file you have to load the pass into `opt`:
```bash
opt -disable-output \
    -load-pass-plugin=build/lib/MyPass.so \
    -passes="my-pass" bar.bc
```

## In-Tree Integration of an Analysis Pass
Integration of an analysis pass into the LLVM source-tree is quite simple with the new PassManager but I had to extract the necessary steps from already existing analyses in the source-tree. There it also seems to be common practice to separate analyses from passes that are actually invoked by e.g. `opt`. Therefore I will follow this approach in this tutorial.

- You have to create two new files:

  Adapted from part two of the [tutorial series](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82) [3] and various (analysis) passes already implemented in LLVM (see `lib/Analysis`):
  - `include/llvm/Analysis/MyAnalysis.h`:
    ```cpp
    #include "llvm/IR/PassManager.h"
    #include "llvm/Passes/PassBuilder.h"
    #include "llvm/Passes/PassPlugin.h"
    #include "llvm/Support/raw_ostream.h"

    #include <iostream>

    namespace llvm {

    // This is the actual analysis that will perform some operation
    class MyAnalysis : public AnalysisInfoMixin<MyAnalysis> {
      // needed so that AnalysisInfoMixin<MyAnalysis> can access
      // private members of MyAnalysis
      friend AnalysisInfoMixin<MyAnalysis>;

      static AnalysisKey Key;

    public:
      // You need to define a result. This can also be some other class.
      using Result = std::string;
      Result run(Function &F, FunctionAnalysisManager &AM);
    };

    // This is the analysis pass that will be invocable via opt
    class MyAnalysisPass : public AnalysisInfoMixin<MyAnalysisPass> {
      raw_ostream &OS;

    public:
      explicit MyAnalysisPass(raw_ostream &OS) : OS(OS) {}
      PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
    };

    } // namespace llvm
    ```

  - `lib/Analysis/MyAnalysis.cpp`:
    ```cpp
    #include "llvm/Analysis/MyAnalysis.h"

    #include "llvm/IR/PassManager.h"
    #include "llvm/Passes/PassBuilder.h"
    #include "llvm/Passes/PassPlugin.h"
    #include "llvm/Support/raw_ostream.h"

    #include <iostream>

    using namespace llvm;

    AnalysisKey MyAnalysis::Key;

    // Definition of the run function of the analysis.
    // Here the actual stuff happens!!!
    std::string MyAnalysis::run(Function &F, FunctionAnalysisManager &FAM) {

      // This pass just iterates over all instructions in all the
      // basic blocks of the function and appends their opcodes to
      // the output string.

      std::string output;
      for (Function::iterator BB = F.begin(); BB != F.end(); BB++) {
        for (BasicBlock::iterator I = BB->begin(); I != BB->end(); I++) {
          output += I->getOpcodeName();
          output += "\n";
        }
      }
      return output;
    }

    // Definition of the run function of the analysis pass that
    // will be invocable via opt. It uses the getResult<Analysis>()
    // method of the FunctionAnalysisManager. This result will be an
    // std::string as we have defined the Result of MyAnalysis above.
    // The result string is piped into the raw_ostream member
    // of the MyAnalysisPass.
    PreservedAnalyses MyAnalysisPass::run(Function &F,
                                          FunctionAnalysisManager &FAM) {
      OS << "Printing analysis results of MyAnalysis for function "
         << "'" << F.getName() << "':"
         << "\n";
      OS << FAM.getResult<MyAnalysis>(F);

      // Analysis should never change the LLVM IR code so all
      // results of other analyses are still valid!
      return PreservedAnalyses::all();
    }
    ```
- Now you have to integrate your analysis pass into the source tree:
  - In `lib/Analysis/CMakeLists.txt` add `MyAnalysis.cpp` to the list of `.cpp` files.

  - In`lib/Passes/PassRegisty.def` add
  ```
  FUNCTION_ANALYSIS("my-analysis", MyAnalysis())
  ```
   to the `FUNCTION_ANALYSIS` section and
   ```
  FUNCTION_PASS("my-analysis-pass", MyAnalysisPass(dbgs()))
  ```
  to the `FUNCTION_PASS` section.

  - In `lib/Passes/PassBuilder.cpp` add `#include "llvm/Analysis/MyAnalysis.h"` to the includes.

Now again you can compile LLVM but this time there will be no need to load a shared library before being able to run the analysis pass:
```bash
opt -passes="my-analysis-pass" bar.bc
```
Note that the string that identifies our `MyAnalysisPass` in the pass pipeline has been defined in `lib/Passes/PassRegisty.def`.

## References
[1] http://releases.llvm.org/6.0.0/docs/GettingStarted.html  
[2] http://releases.llvm.org/6.0.0/docs/WritingAnLLVMPass.html  
[3] https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82
