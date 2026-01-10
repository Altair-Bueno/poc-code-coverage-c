```sh
cmake -S . -B build \
    -DENABLE_COVERAGE=ON \
    -DBUILD_TESTING=ON \
    -DCMAKE_C_COMPILER=clang \
    -DLLVM_PROFDATA=`xcrun --find llvm-profdata` \
    -DLLVM_COV=`xcrun --find llvm-cov`
cmake --build build
ctest --test-dir build
cmake --build build --target coverage
```
