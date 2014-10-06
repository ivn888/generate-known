This guide describes in depth generation of source files for CMake targets, some pitfalls and optimizations. In this example python script `script.py` creates source files `A1.cpp` and `A2.cpp`. This files used to create library target `A`. Example starts with simple structure, more complex code can be found in other branches. Executable target `foo` used to test the fact that changes applied (more libraries will be linked further). This executable driven by test `RunFoo`, run it by `ctest -VV` (`-VV` to see test output). Some experiments below change code, please revert everything back before next test (`git checkout -- *`, `rm -rf _builds`).

Important to note that:
* we know that `script.py` will create only known files (i.e. source list is fixed: `A1.cpp`, `A2.cpp`)
* there are only one target that use generated source files
* custom command and target is in the same directory

![origin][3]

## Try it

```bash
> cmake -H. -B_builds
> cmake --build _builds
```

Result:
```
[ 25%] Generate (custom command)
Generate (python script)
Scanning dependencies of target A
[ 50%] Building CXX object CMakeFiles/A.dir/generated/A1.cpp.o
[ 75%] Building CXX object CMakeFiles/A.dir/generated/A2.cpp.o
Linking CXX static library libA.a
[ 75%] Built target A
Scanning dependencies of target foo
[100%] Building CXX object CMakeFiles/foo.dir/main.cpp.o
Linking CXX executable foo.exe
[100%] Built target foo
```

Run again (no changes):
```bash
> cmake --build _builds
[ 75%] Built target A
[100%] Built target foo
```

Test it:
```bash
> (cd _builds && ctest -VV)
...
1: A1 say: Hello from A1
1: A2 say: Hello from A2
...
```

Change python script so that `A1.cpp` emits other message (important to change the script so that generated result `A1.cpp` will change, not only python code, see details below). Note that only one file rebuild:
```bash
> grep new script.py
  return "Hello from A1 (new)";
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script)
Scanning dependencies of target A
[ 50%] Building CXX object CMakeFiles/A.dir/generated/A1.cpp.o
Linking CXX static library libA.a
[ 75%] Built target A
Linking CXX executable foo.exe
[100%] Built target foo
```

Run again (no changes):
```
> cmake --build _builds
[ 75%] Built target A
[100%] Built target foo
```

Test new message:
```bash
> (cd _builds && ctest -VV)
...
1: A1 say: Hello from A1 (new)
1: A2 say: Hello from A2
...
```

## Details

### Out-of-source

Follow [out-of-source][1] concept and create generated files inside build directory:

```cmake
set(gen_dir "${PROJECT_BINARY_DIR}/generated")
# Files: "${gen_dir}/A1.cpp", "${gen_dir}/A2.cpp"
```

### add_custom_command

* Use [add_custom_command][2] to create `A{1,2}.cpp` files:

```cmake
add_custom_command(
    OUTPUT "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp"
    COMMAND
        "${PYTHON_EXECUTABLE}"
        "${CMAKE_CURRENT_LIST_DIR}/script.py"
        --dir "${gen_dir}"
        --smart
        --check-changes
    DEPENDS "${CMAKE_CURRENT_LIST_DIR}/script.py"
    COMMENT "Generate (custom command)"
)
```

* `OUTPUT "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp"` said that CMake expected this list to be generated by `COMMAND`. Files from this list will be automatically marked as [GENERATED][4]. For example compare `main.cpp` and `A1.cpp` properties:

```cmake
# Add this code to the end of CMakeLists.txt
get_source_file_property(main_gen main.cpp GENERATED)
get_source_file_property(A1_gen "${gen_dir}/A1.cpp" GENERATED)

message("main: ${main_gen}, A1: ${A1_gen}")
```

```bash
> cmake -H. -B_builds
...
main: NOTFOUND, A1: 1
...
```

* Property [GENERATED][4] used to avoid error `Cannot find source file`:
```cmake
# Remove A2.cpp from OUTPUT list
add_custom_command(
    OUTPUT "${gen_dir}/A1.cpp"
    COMMAND
...

# Now command add_library expect file "${gen_dir}/A2.cpp"
# but there is no such file (it's not generated yet)
add_library(A "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp")
```

```bash
> cmake -H. -B_builds
-- Configuring done
CMake Error at CMakeLists.txt:21 (add_library):
  Cannot find source file:

    /.../_builds/generated/A2.cpp

  Tried extensions .c .C .c++ .cc .cpp .cxx .m .M .mm .h .hh .h++ .hm .hpp
  .hxx .in .txx
```

* Also if any target from this directory use this files, CMake automatically set dependency `target` -> `custom_command`, see [documentation][2]:

```
A target created in the same directory (CMakeLists.txt file) that specifies
any output of the custom command as a source file is given a rule to
generate the file using the command at build time
```

If you remove all target that use `OUTPUT` then custom command will not be invoked, for example lets remove library `A`:

![nodeps][TODO]

```cmake
# CMakeLists.txt
# add_library(A "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp")

add_executable(foo main.cpp)
# target_link_libraries(foo A)
```

```
// main.cpp
int main() {
//  std::cout << "A1 say: " << A1() << std::endl;
//  std::cout << "A2 say: " << A2() << std::endl;
}
```

```bash
> cmake -H. -B_builds
> cmake --build _builds
```

No custom command in build:
```
Scanning dependencies of target foo
[100%] Building CXX object CMakeFiles/foo.dir/main.cpp.o
Linking CXX executable foo.exe
[100%] Built target foo
```

* `DEPENDS "${CMAKE_CURRENT_LIST_DIR}/script.py"` show that if `script.py` file changes then `COMMAND` must be re-run. If this line will be removed custom command will not be run until files `A1.cpp` and `A2.cpp` exist. New custom command:
```cmake
add_custom_command(
    OUTPUT "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp"
    COMMAND
        "${PYTHON_EXECUTABLE}"
        "${CMAKE_CURRENT_LIST_DIR}/script.py"
        --dir "${gen_dir}"
        --smart
        --check-changes
#    DEPENDS "${CMAKE_CURRENT_LIST_DIR}/script.py"
    COMMENT "Generate (custom command)"
)
```

```bash
> cmake -H. -B_builds
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script)
Scanning dependencies of target A
[ 50%] Building CXX object CMakeFiles/A.dir/generated/A1.cpp.o
[ 75%] Building CXX object CMakeFiles/A.dir/generated/A2.cpp.o
Linking CXX static library libA.a
[ 75%] Built target A
Scanning dependencies of target foo
[100%] Building CXX object CMakeFiles/foo.dir/main.cpp.o
Linking CXX executable foo.exe
[100%] Built target foo
```

Modify `script.py`:

```bash
> grep new script.py
  return "Hello from A1 (new)";
```

Change will be ignored:

```bash
> cmake --build _builds
[ 75%] Built target A
[100%] Built target foo
```

One of the files `A1.cpp` or `A2.cpp` need to be removed to run custom command again:

```bash
> rm -rf _builds/generated/A1.cpp
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script)
...
```

* `--smart` option used to avoid "touching" generated files that in fact have no changes. This allow to rebuild only files that really needed. Remove `--smart` and see that any modification of `script.py` will affect both `A1.cpp` and `A2.cpp`:

```cmake
add_custom_command(
    OUTPUT "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp"
    COMMAND
        "${PYTHON_EXECUTABLE}"
        "${CMAKE_CURRENT_LIST_DIR}/script.py"
        --dir "${gen_dir}"
#        --smart
        --check-changes
    DEPENDS "${CMAKE_CURRENT_LIST_DIR}/script.py"
    COMMENT "Generate (custom command)"
)
```

Build it first time:
```bash
> cmake -H. -B_builds
> cmake --build _builds
```

Change that not really touches any of `A{1,2}.cpp`:
```
> grep new script.py
print('Generate (python script, new)')
```

Re-run, note that both `A1.cpp` and `A2.cpp` rebuilt (but not `main.cpp`):

```bash
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script, new)
Scanning dependencies of target A
[ 50%] Building CXX object CMakeFiles/A.dir/generated/A1.cpp.o
[ 75%] Building CXX object CMakeFiles/A.dir/generated/A2.cpp.o
Linking CXX static library libA.a
[ 75%] Built target A
Linking CXX executable foo.exe
[100%] Built target foo
```

* Note that [dependency][2] has form:
```
OUTPUT: DEPENDS
    COMMAND
```

Applying to current case:
```
A1.cpp A2.cpp: script.py
  python script.py ...
```

It means that if `script.py` changes, but `A1.cpp` or `A2.cpp` not, then custom command will be run every time.
Remove `--check-changes`:

```
add_custom_command(
    OUTPUT "${gen_dir}/A1.cpp" "${gen_dir}/A2.cpp"
    COMMAND
        "${PYTHON_EXECUTABLE}"
        "${CMAKE_CURRENT_LIST_DIR}/script.py"
        --dir "${gen_dir}"
        --smart
#        --check-changes
    DEPENDS "${CMAKE_CURRENT_LIST_DIR}/script.py"
    COMMENT "Generate (custom command)"
)
```

Build project:
```bash
> cmake -H. -B_builds
> cmake --build _builds
```

Modify `script.py`:
```
> grep new script.py
print('Generate (python script, new)')
```

Now custom command run every time again:
```
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script, new)
[ 25%] Generate (custom command)
Generate (python script, new)
[ 75%] Built target A
[100%] Built target foo
```

... and again:
```
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script, new)
[ 25%] Generate (custom command)
Generate (python script, new)
[ 75%] Built target A
[100%] Built target foo
```

If you put this option back, then script will report that flaw:
```bash
> cmake --build _builds
[ 25%] Generate (custom command)
Generate (python script)
No changes!
CMakeFiles/A.dir/build.make:53: recipe for target 'generated/A1.cpp' failed
...
make: *** [all] Error 2
```

You can use any strategy here, for example touch only one generated file that has minimal recompilation time.

[1]: http://www.cmake.org/Wiki/CMake_FAQ#Out-of-source_build_trees
[2]: http://www.cmake.org/cmake/help/v3.0/command/add_custom_command.html
[3]: https://raw.githubusercontent.com/forexample/generate-known/master/diagrams/origin.png
[4]: http://www.cmake.org/cmake/help/v3.0/prop_sf/GENERATED.html
