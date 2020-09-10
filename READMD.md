# Fault Injector (LLVM 10.0.1)

## Build

```
git clone https://github.com/HPCL-INHA/fault-injector
cd fault-injector
mkdir build
cd build
cmake ../
cmake --build tools/opt -- -j32
cmake --build tools/llc -- -j32
```

## How to use

### Fault Injector Legacy

```
Copy files in test directory to your test codes

clang -S -emit-llvm test.c
./opt -faultinjectlegacy test.ll -S -o test-injected.ll
llc -filetype=obj test-injected.ll -o test-injected.obj

clang -S -emit-llvm rtl-core.c
llc -filetype=obj rtl-core.ll -o rtl-core.obj

clang rtl-core.obj test-injected.obj
```

### Fault Injector to Minimize the Impact of Binary Code

```
import sys
import os
import os.path
from subprocess import Popen, PIPE


if len(sys.argv) == 1:
  print "LLVM Based FaultInjector"
  print "run.py <Code File .c or .cpp>"
  exit(1)

if os.path.isfile(sys.argv[1]) == False:
  print "no such file or directory: " + sys.argv[1]
  exit(1)
  
clang_addr = "clang"
opt_addr = "../opt"              <= Set your opt binary path
llc_addr = "../llc"                 <= Set your llc binary path

filename = os.path.splitext(sys.argv[1])[0]
clang_result = filename + "-1-clang.ll"
process = Popen([clang_addr, sys.argv[1], "-S", "-emit-llvm", "-o", clang_result, "-mllvm", "-disable-llvm-optzns", "-disable-llvm-passes", "-O1"])
process.wait()

# Run faultinject pass
opt_faultinject = filename + "-2-faultinject.ll"
process = Popen([opt_addr, "-S", clang_result, "-o", opt_faultinject, "-faultinject"])
process.wait()

# Run O2 optimizing
o2_result = filename + "-3-o2.ll"
process = Popen([opt_addr, "-S", opt_faultinject, "-o", o2_result, "-O2"])
process.wait()

# 1. Run injectfault
if_result = filename + "-4-injectfault.ll"
process = Popen([opt_addr, "-S", o2_result, "-o", if_result, "-injectfault"])
process.wait()

# 2. Run deleteinjectfunc
dif_result = filename + "-4-deleteinjectfault.ll"
process = Popen([opt_addr, "-S", o2_result, "-o", dif_result, "-deleteinjectfunc"])
process.wait()

# 1-1. Run llc
llc_1 = filename + "-5-llc-injectfault.s"
process = Popen([llc_addr, if_result, "-o", llc_1])
process.wait()

# 2-1. Run llc
llc_2 = filename + "-5-llc-none.s"
process = Popen([llc_addr, dif_result, "-o", llc_2])
process.wait()

# 1-3. Extract binary
obj_name1 = filename + "-6-bin-injectfault.obj"
bin_name1 = filename + "-6-bin-injectfault"
process = Popen([llc_addr, "-filetype=obj", if_result, "-o", obj_name1])
process.wait()
process = Popen([clang_addr, obj_name1, "-o", bin_name1])
process.wait()

# 2-3. Extract binary
obj_name2 = filename + "-6-bin-none.obj"
bin_name2 = filename + "-6-bin-none"
process = Popen([llc_addr, "-filetype=obj", dif_result, "-o", obj_name2])
process.wait()
process = Popen([clang_addr, obj_name2, "-o", bin_name2])
process.wait()
```