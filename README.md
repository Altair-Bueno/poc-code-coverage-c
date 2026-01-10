1. Configure the project to be build with code coverage enabled and tests

> [INFO] If you are running on macOS, you may need to specify the paths to `llvm-profdata` and `llvm-cov` using `xcrun --find` as seen below.

```sh
cmake -S . -B build \
    -DENABLE_COVERAGE=ON \
    -DBUILD_TESTING=ON \
    -DCMAKE_C_COMPILER=clang \
    -DLLVM_PROFDATA=`xcrun --find llvm-profdata` \
    -DLLVM_COV=`xcrun --find llvm-cov`
```

2. Build the project

```sh
cmake --build build
```

3. Run the tests

```sh
ctest --test-dir build
```

4. Generate the code coverage report. It will available at `build/coverage/index.html`

```sh
cmake --build build --target coverage
```
