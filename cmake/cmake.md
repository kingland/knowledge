# CMake: Modern Build System Guide

CMake is a cross-platform build system generator that creates native build files (Makefiles, Visual Studio projects, Xcode projects, etc.) from simple configuration files.

## Basic Usage

### 1. **Simple Project Structure**
```
project/
├── CMakeLists.txt
├── src/
│   └── main.cpp
└── include/
    └── myheader.h
```

### 2. **Minimal CMakeLists.txt**
```cmake
cmake_minimum_required(VERSION 3.15)
project(MyApp VERSION 1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(myapp src/main.cpp)
target_include_directories(myapp PRIVATE include)
```

### 3. **Build Process**
```bash
# Out-of-source build (recommended)
mkdir build && cd build
cmake ..              # Generate build files
cmake --build .       # Build the project

# Or with make
make

# Install
cmake --install .
```

## Main Features

### **1. Cross-Platform Support**
Generates native build systems for different platforms:
- Linux/macOS: Makefiles, Ninja
- Windows: Visual Studio, MinGW
- Xcode projects for macOS/iOS

### **2. Target-Based System**
Modern CMake uses targets rather than variables:

```cmake
# Create library target
add_library(mylib STATIC src/lib.cpp)
target_include_directories(mylib PUBLIC include)
target_compile_features(mylib PUBLIC cxx_std_17)

# Create executable target
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

### **3. Dependency Management**

**Find existing packages:**
```cmake
find_package(Boost 1.70 REQUIRED COMPONENTS filesystem system)
target_link_libraries(myapp PRIVATE Boost::filesystem Boost::system)
```

**FetchContent for external dependencies:**
```cmake
include(FetchContent)
FetchContent_Declare(
    json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG v3.11.2
)
FetchContent_MakeAvailable(json)
target_link_libraries(myapp PRIVATE nlohmann_json::nlohmann_json)
```

### **4. Build Configurations**
```cmake
# Debug/Release configurations
cmake -DCMAKE_BUILD_TYPE=Release ..

# Custom configurations
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(myapp PRIVATE DEBUG_MODE)
endif()
```

### **5. Compiler Options Management**
```cmake
# Warning flags
target_compile_options(myapp PRIVATE
    $<$<CXX_COMPILER_ID:GNU>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# Compile definitions
target_compile_definitions(myapp PRIVATE
    $<$<CONFIG:Debug>:DEBUG_LOGGING>
)
```

### **6. Testing Support**
```cmake
enable_testing()
add_executable(test_myapp tests/test.cpp)
target_link_libraries(test_myapp PRIVATE mylib)

add_test(NAME MyTest COMMAND test_myapp)

# Run tests
# ctest or cmake --build . --target test
```

### **7. Installation Rules**
```cmake
install(TARGETS myapp mylib
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

install(DIRECTORY include/ DESTINATION include)
install(FILES LICENSE README.md DESTINATION share/doc/myapp)
```

### **8. Generator Expressions**
Powerful conditional logic evaluated at build-time:
```cmake
target_compile_definitions(myapp PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>
    $<$<PLATFORM_ID:Windows>:WINDOWS_BUILD>
)
```

### **9. Interface Libraries** (Header-only)
```cmake
add_library(myheaderlib INTERFACE)
target_include_directories(myheaderlib INTERFACE include)
target_compile_features(myheaderlib INTERFACE cxx_std_20)
```

### **10. Custom Commands**
```cmake
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/generated.cpp
    COMMAND python ${CMAKE_SOURCE_DIR}/codegen.py
    DEPENDS ${CMAKE_SOURCE_DIR}/config.xml
)
```

## Complete Example

```cmake
cmake_minimum_required(VERSION 3.15)
project(CompleteApp VERSION 1.0.0 LANGUAGES CXX)

# Options
option(BUILD_TESTS "Build tests" ON)
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)

# C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Dependencies
find_package(Threads REQUIRED)
find_package(Boost 1.70 REQUIRED COMPONENTS filesystem)

# Library target
add_library(mylib
    src/core.cpp
    src/utils.cpp
)

target_include_directories(mylib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

target_link_libraries(mylib
    PUBLIC Boost::filesystem
    PRIVATE Threads::Threads
)

# Executable target
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib)

# Tests
if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

# Installation
install(TARGETS myapp mylib
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)
```

## Key Advantages

1. **Modern approach**: Target-based instead of directory-based
2. **Transitive dependencies**: Automatically propagated through targets
3. **Generator expressions**: Build-time conditional logic
4. **Package management**: Find and integrate external libraries easily
5. **IDE integration**: Generates native project files
6. **Toolchain files**: Cross-compilation support

CMake has become the de facto standard for C++ projects due to its flexibility and powerful dependency management capabilities.

# CMake Global Variables

CMake has many built-in global variables that control the build process. Here are the most important ones:

## 1. Project Information Variables

```cmake
# Set by project() command
${PROJECT_NAME}              # Current project name
${PROJECT_VERSION}           # Project version
${PROJECT_SOURCE_DIR}        # Top-level source directory
${PROJECT_BINARY_DIR}        # Top-level build directory

# CMAKE prefix versions (always refer to top-most project)
${CMAKE_PROJECT_NAME}        # Top-most project name
${CMAKE_SOURCE_DIR}          # Top-most source directory
${CMAKE_BINARY_DIR}          # Top-most build directory
${CMAKE_CURRENT_SOURCE_DIR}  # Current CMakeLists.txt directory
${CMAKE_CURRENT_BINARY_DIR}  # Current build directory
```

**Example:**
```cmake
project(MyApp VERSION 1.2.3)

message("Project: ${PROJECT_NAME}")           # MyApp
message("Version: ${PROJECT_VERSION}")        # 1.2.3
message("Source: ${PROJECT_SOURCE_DIR}")      # /home/user/myapp
message("Build: ${PROJECT_BINARY_DIR}")       # /home/user/myapp/build
```

## 2. Compiler and Build Type Variables

```cmake
# Compiler identification
${CMAKE_CXX_COMPILER}        # C++ compiler path
${CMAKE_C_COMPILER}          # C compiler path
${CMAKE_CXX_COMPILER_ID}     # GNU, Clang, MSVC, etc.
${CMAKE_CXX_COMPILER_VERSION}

# Build configuration
${CMAKE_BUILD_TYPE}          # Debug, Release, RelWithDebInfo, MinSizeRel

# Standard versions
${CMAKE_CXX_STANDARD}        # 11, 14, 17, 20, 23
${CMAKE_CXX_STANDARD_REQUIRED}
${CMAKE_CXX_EXTENSIONS}      # ON/OFF for compiler extensions
```

**Example:**
```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    message("Building in Debug mode")
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # GCC-specific options
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    # MSVC-specific options
endif()
```

## 3. Platform Detection Variables

```cmake
# Operating System
${CMAKE_SYSTEM_NAME}         # Linux, Windows, Darwin (macOS), etc.
${WIN32}                     # TRUE on Windows
${UNIX}                      # TRUE on Unix-like systems
${APPLE}                     # TRUE on macOS
${LINUX}                     # TRUE on Linux
${MSVC}                      # TRUE for Microsoft Visual C++
${MINGW}                     # TRUE for MinGW

# Architecture
${CMAKE_SIZEOF_VOID_P}       # 4 for 32-bit, 8 for 64-bit
```

**Example:**
```cmake
if(WIN32)
    message("Windows platform")
    set(PLATFORM_LIBS ws2_32 wsock32)
elseif(UNIX)
    message("Unix platform")
    set(PLATFORM_LIBS pthread dl)
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    message("64-bit build")
else()
    message("32-bit build")
endif()
```

## 4. Compilation Flags Variables

```cmake
# Compiler flags
${CMAKE_CXX_FLAGS}                    # General C++ flags
${CMAKE_CXX_FLAGS_DEBUG}              # Debug-specific flags
${CMAKE_CXX_FLAGS_RELEASE}            # Release-specific flags
${CMAKE_CXX_FLAGS_RELWITHDEBINFO}     # Release with debug info
${CMAKE_CXX_FLAGS_MINSIZEREL}         # Minimum size release

${CMAKE_C_FLAGS}                      # C flags variants

# Linker flags
${CMAKE_EXE_LINKER_FLAGS}             # Executable linker flags
${CMAKE_SHARED_LINKER_FLAGS}          # Shared library linker flags
${CMAKE_STATIC_LINKER_FLAGS}          # Static library linker flags
```

**Example:**
```cmake
# Add flags globally (not recommended)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -O0")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")

# Better: use target-specific
add_executable(myapp main.cpp)
target_compile_options(myapp PRIVATE -Wall -Wextra)
```

## 5. Output Directory Variables

```cmake
# Default output directories
${CMAKE_RUNTIME_OUTPUT_DIRECTORY}    # Executables
${CMAKE_LIBRARY_OUTPUT_DIRECTORY}    # Shared libraries (.so, .dll)
${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}    # Static libraries (.a, .lib)

# Configuration-specific
${CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG}
${CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE}
```

**Example:**
```cmake
# Put all executables in bin/
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Put all libraries in lib/
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
```

## 6. Installation Variables

```cmake
${CMAKE_INSTALL_PREFIX}              # Installation root (/usr/local default)
${CMAKE_INSTALL_BINDIR}              # bin
${CMAKE_INSTALL_LIBDIR}              # lib or lib64
${CMAKE_INSTALL_INCLUDEDIR}          # include
${CMAKE_INSTALL_DATADIR}             # share
```

**Example:**
```cmake
include(GNUInstallDirs)

install(TARGETS myapp
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# Change install prefix
# cmake -DCMAKE_INSTALL_PREFIX=/opt/myapp ..
```

## 7. Module and Package Variables

```cmake
${CMAKE_MODULE_PATH}         # Search path for Find*.cmake modules
${CMAKE_PREFIX_PATH}         # Search path for find_package()
```

**Example:**
```cmake
# Add custom module directory
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")

# Add custom package search path
list(APPEND CMAKE_PREFIX_PATH "/opt/mylibs")

find_package(MyCustomLib REQUIRED)
```

## 8. Cache Variables

```cmake
# Variables that persist between CMake runs
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_TESTING "Build tests" OFF)

set(MY_OPTION "value" CACHE STRING "Description")
```

**Example:**
```cmake
option(ENABLE_TESTS "Enable testing" ON)
option(ENABLE_LOGGING "Enable logging" OFF)

if(ENABLE_TESTS)
    enable_testing()
endif()

# Set from command line: cmake -DENABLE_LOGGING=ON ..
```

## 9. Generator Variables

```cmake
${CMAKE_GENERATOR}           # Unix Makefiles, Ninja, Visual Studio, etc.
${CMAKE_MAKE_PROGRAM}        # make, ninja, msbuild, etc.
```

**Example:**
```cmake
message("Generator: ${CMAKE_GENERATOR}")

if(CMAKE_GENERATOR STREQUAL "Ninja")
    message("Using Ninja build system")
endif()
```

## 10. Version Variables

```cmake
${CMAKE_VERSION}             # CMake version
${CMAKE_MAJOR_VERSION}
${CMAKE_MINOR_VERSION}
${CMAKE_PATCH_VERSION}
```

## Complete Example Using Global Variables

```cmake
cmake_minimum_required(VERSION 3.15)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

# Display information
message(STATUS "Project: ${PROJECT_NAME} ${PROJECT_VERSION}")
message(STATUS "System: ${CMAKE_SYSTEM_NAME}")
message(STATUS "Compiler: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# Build options
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(ENABLE_TESTS "Build tests" ON)

# Platform-specific settings
if(WIN32)
    add_definitions(-DWINDOWS_BUILD)
    set(PLATFORM_LIBS ws2_32)
elseif(UNIX)
    add_definitions(-DUNIX_BUILD)
    set(PLATFORM_LIBS pthread dl)
endif()

# Compiler-specific flags
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
endif()

# Debug/Release flags
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")

# Module path
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

# Create executable
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE ${PLATFORM_LIBS})

# Testing
if(ENABLE_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

# Installation
include(GNUInstallDirs)
install(TARGETS myapp
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

## Setting Variables from Command Line

```bash
# Set build type
cmake -DCMAKE_BUILD_TYPE=Release ..

# Set install prefix
cmake -DCMAKE_INSTALL_PREFIX=/opt/myapp ..

# Set custom option
cmake -DENABLE_TESTS=ON ..

# Set compiler
cmake -DCMAKE_CXX_COMPILER=clang++ ..

# Multiple variables
cmake -DCMAKE_BUILD_TYPE=Debug -DENABLE_TESTS=ON -DBUILD_SHARED_LIBS=ON ..
```

## Best Practices

1. **Avoid modifying global flag variables** like `CMAKE_CXX_FLAGS` directly - use `target_compile_options()` instead
2. **Use `CMAKE_CURRENT_*` variables** when working in subdirectories
3. **Check variable existence** before using: `if(DEFINED MY_VAR)`
4. **Use cache variables** for user-configurable options
5. **Prefer target properties** over global variables for modern CMake

These global variables give you fine control over the build process while maintaining cross-platform compatibility!

# CMake Control Statements

CMake has various control flow statements for conditional logic, loops, and flow control.

## 1. Conditional Statements (if/elseif/else)

### **Basic if Statement**

```cmake
if(condition)
    # statements
endif()
```

### **if/else**

```cmake
if(condition)
    # statements when true
else()
    # statements when false
endif()
```

### **if/elseif/else**

```cmake
if(condition1)
    # statements for condition1
elseif(condition2)
    # statements for condition2
elseif(condition3)
    # statements for condition3
else()
    # statements when all false
endif()
```

### **Condition Types**

```cmake
# Boolean constants
if(TRUE)
if(FALSE)
if(ON)
if(OFF)
if(YES)
if(NO)

# Variable existence and value
if(DEFINED MY_VAR)              # Variable is defined
if(MY_VAR)                      # Variable is true-like
if(NOT MY_VAR)                  # Variable is false-like

# String comparison
if(MY_VAR STREQUAL "value")     # String equality
if(MY_VAR MATCHES "regex")      # Regex match
if(MY_VAR LESS "value")         # Alphabetical less than
if(MY_VAR GREATER "value")      # Alphabetical greater than

# Numeric comparison
if(MY_VAR EQUAL 5)              # Numeric equality
if(MY_VAR LESS 10)              # Less than
if(MY_VAR GREATER 5)            # Greater than
if(MY_VAR LESS_EQUAL 10)        # Less than or equal
if(MY_VAR GREATER_EQUAL 5)      # Greater than or equal

# Version comparison
if(MY_VERSION VERSION_EQUAL "1.2.3")
if(MY_VERSION VERSION_LESS "2.0.0")
if(MY_VERSION VERSION_GREATER "1.0.0")
if(MY_VERSION VERSION_LESS_EQUAL "1.5.0")
if(MY_VERSION VERSION_GREATER_EQUAL "1.2.0")

# Logical operators
if(NOT condition)               # Logical NOT
if(cond1 AND cond2)            # Logical AND
if(cond1 OR cond2)             # Logical OR
if((cond1 OR cond2) AND cond3) # Grouping with parentheses

# File operations
if(EXISTS "/path/to/file")      # File or directory exists
if(IS_DIRECTORY "/path")        # Is a directory
if(IS_SYMLINK "/path")          # Is a symbolic link
if(IS_ABSOLUTE "/path")         # Is absolute path

# Target operations
if(TARGET mylib)                # Target exists
```

### **Practical Examples**

```cmake
# Check build type
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DDEBUG_MODE)
    message("Building in Debug mode")
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(-DRELEASE_MODE)
    message("Building in Release mode")
else()
    message("Unknown build type")
endif()

# Platform detection
if(WIN32)
    set(PLATFORM_SOURCES src/windows.cpp)
elseif(UNIX AND NOT APPLE)
    set(PLATFORM_SOURCES src/linux.cpp)
elseif(APPLE)
    set(PLATFORM_SOURCES src/macos.cpp)
endif()

# Compiler detection
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    target_compile_options(myapp PRIVATE -Wall -Wextra)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_compile_options(myapp PRIVATE -Wall -Weverything)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    target_compile_options(myapp PRIVATE /W4)
endif()

# Version check
if(CMAKE_VERSION VERSION_LESS "3.15")
    message(FATAL_ERROR "CMake 3.15 or higher required")
endif()

# Check if file exists
if(EXISTS "${CMAKE_SOURCE_DIR}/config.txt")
    message("Config file found")
else()
    message("Config file not found")
endif()

# Complex conditions
if((BUILD_TESTS AND EXISTS "${CMAKE_SOURCE_DIR}/tests") OR FORCE_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

## 2. Loops

### **foreach Loop**

```cmake
# Iterate over list
foreach(item IN LISTS my_list)
    message("Item: ${item}")
endforeach()

# Iterate over items directly
foreach(item apple banana cherry)
    message("Fruit: ${item}")
endforeach()

# Iterate with range
foreach(i RANGE 5)          # 0 to 5
    message("Number: ${i}")
endforeach()

foreach(i RANGE 1 10)       # 1 to 10
    message("Number: ${i}")
endforeach()

foreach(i RANGE 0 10 2)     # 0 to 10, step 2
    message("Even: ${i}")
endforeach()

# Iterate over multiple lists (ZIP)
foreach(var1 var2 IN ZIP_LISTS list1 list2)
    message("${var1} - ${var2}")
endforeach()
```

### **while Loop**

```cmake
set(counter 0)
while(counter LESS 5)
    message("Counter: ${counter}")
    math(EXPR counter "${counter} + 1")
endwhile()

# Infinite loop with break
while(TRUE)
    # some condition
    if(done)
        break()
    endif()
endwhile()
```

### **Loop Control**

```cmake
# break() - exit loop
foreach(i RANGE 10)
    if(i EQUAL 5)
        break()
    endif()
    message("i = ${i}")
endforeach()

# continue() - skip to next iteration
foreach(i RANGE 10)
    if(i EQUAL 5)
        continue()
    endif()
    message("i = ${i}")  # Won't print 5
endforeach()
```

### **Practical Loop Examples**

```cmake
# Compile multiple source files
set(SOURCES main.cpp utils.cpp helper.cpp)
foreach(src IN LISTS SOURCES)
    message("Adding source: ${src}")
endforeach()
add_executable(myapp ${SOURCES})

# Add compiler flags
set(WARNING_FLAGS -Wall -Wextra -Wpedantic -Werror)
foreach(flag IN LISTS WARNING_FLAGS)
    target_compile_options(myapp PRIVATE ${flag})
endforeach()

# Process multiple subdirectories
set(MODULES core ui network database)
foreach(module IN LISTS MODULES)
    if(EXISTS "${CMAKE_SOURCE_DIR}/${module}")
        add_subdirectory(${module})
    endif()
endforeach()

# Find all source files
file(GLOB_RECURSE SOURCE_FILES "src/*.cpp")
foreach(source IN LISTS SOURCE_FILES)
    message("Found source: ${source}")
endforeach()

# Create multiple executables
set(APPS app1 app2 app3)
foreach(app IN LISTS APPS)
    add_executable(${app} src/${app}.cpp)
    target_link_libraries(${app} PRIVATE common_lib)
endforeach()
```

## 3. Function and Macro

### **Function** (has own scope)

```cmake
function(my_function arg1 arg2)
    message("Arg1: ${arg1}")
    message("Arg2: ${arg2}")
    
    # Local variables don't affect parent scope
    set(local_var "value")
    
    # Use PARENT_SCOPE to affect parent
    set(result "output" PARENT_SCOPE)
endfunction()

# Call function
my_function("hello" "world")
```

### **Macro** (no own scope)

```cmake
macro(my_macro arg1 arg2)
    message("Arg1: ${arg1}")
    message("Arg2: ${arg2}")
    
    # Variables affect parent scope directly
    set(result "output")
endmacro()

# Call macro
my_macro("hello" "world")
```

### **Function with Named Arguments**

```cmake
function(add_my_library)
    set(options OPTIONAL SHARED STATIC)
    set(oneValueArgs NAME VERSION)
    set(multiValueArgs SOURCES DEPENDENCIES)
    
    cmake_parse_arguments(
        ARG
        "${options}"
        "${oneValueArgs}"
        "${multiValueArgs}"
        ${ARGN}
    )
    
    message("Library name: ${ARG_NAME}")
    message("Version: ${ARG_VERSION}")
    message("Sources: ${ARG_SOURCES}")
    message("Dependencies: ${ARG_DEPENDENCIES}")
    
    if(ARG_SHARED)
        add_library(${ARG_NAME} SHARED ${ARG_SOURCES})
    else()
        add_library(${ARG_NAME} STATIC ${ARG_SOURCES})
    endif()
    
    target_link_libraries(${ARG_NAME} PRIVATE ${ARG_DEPENDENCIES})
endfunction()

# Usage
add_my_library(
    NAME mylib
    VERSION 1.0
    SOURCES src/a.cpp src/b.cpp
    DEPENDENCIES Boost::filesystem pthread
    SHARED
)
```

### **Practical Function Examples**

```cmake
# Function to add executable with common settings
function(add_my_executable name)
    add_executable(${name} ${ARGN})
    target_compile_features(${name} PRIVATE cxx_std_17)
    target_compile_options(${name} PRIVATE
        $<$<CXX_COMPILER_ID:GNU>:-Wall -Wextra>
        $<$<CXX_COMPILER_ID:MSVC>:/W4>
    )
endfunction()

add_my_executable(app1 src/app1.cpp)
add_my_executable(app2 src/app2.cpp src/utils.cpp)

# Function to print all variables
function(print_all_variables)
    get_cmake_property(vars VARIABLES)
    foreach(var ${vars})
        message("${var} = ${${var}}")
    endforeach()
endfunction()

# Function with return value
function(get_platform_name result)
    if(WIN32)
        set(${result} "Windows" PARENT_SCOPE)
    elseif(APPLE)
        set(${result} "macOS" PARENT_SCOPE)
    elseif(UNIX)
        set(${result} "Linux" PARENT_SCOPE)
    else()
        set(${result} "Unknown" PARENT_SCOPE)
    endif()
endfunction()

get_platform_name(platform)
message("Platform: ${platform}")
```

## 4. Return Statement

```cmake
function(check_condition)
    if(NOT SOME_CONDITION)
        message("Condition not met, returning early")
        return()
    endif()
    
    # Continue processing
    message("Processing...")
endfunction()

# Return with propagate
function(early_return)
    if(ERROR_CONDITION)
        return(PROPAGATE var1 var2)  # Propagate variables to parent
    endif()
endfunction()
```

## Complete Practical Example

```cmake
cmake_minimum_required(VERSION 3.15)
project(ControlFlowExample)

# Options
option(BUILD_TESTS "Build tests" OFF)
option(BUILD_EXAMPLES "Build examples" ON)
option(ENABLE_WARNINGS "Enable warnings" ON)

# Function to configure target
function(configure_target target_name)
    # Set C++ standard
    target_compile_features(${target_name} PRIVATE cxx_std_17)
    
    # Add warnings if enabled
    if(ENABLE_WARNINGS)
        if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR 
           CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
            target_compile_options(${target_name} PRIVATE
                -Wall -Wextra -Wpedantic
            )
        elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
            target_compile_options(${target_name} PRIVATE /W4)
        endif()
    endif()
    
    # Platform-specific settings
    if(WIN32)
        target_compile_definitions(${target_name} PRIVATE PLATFORM_WINDOWS)
    elseif(APPLE)
        target_compile_definitions(${target_name} PRIVATE PLATFORM_MACOS)
    elseif(UNIX)
        target_compile_definitions(${target_name} PRIVATE PLATFORM_LINUX)
    endif()
endfunction()

# Main library
set(LIB_SOURCES
    src/core.cpp
    src/utils.cpp
    src/parser.cpp
)

add_library(mylib STATIC ${LIB_SOURCES})
configure_target(mylib)

# Main executable
add_executable(myapp src/main.cpp)
configure_target(myapp)
target_link_libraries(myapp PRIVATE mylib)

# Build examples if enabled
if(BUILD_EXAMPLES)
    set(EXAMPLES example1 example2 example3)
    foreach(example IN LISTS EXAMPLES)
        if(EXISTS "${CMAKE_SOURCE_DIR}/examples/${example}.cpp")
            add_executable(${example} examples/${example}.cpp)
            configure_target(${example})
            target_link_libraries(${example} PRIVATE mylib)
        else()
            message(WARNING "Example ${example}.cpp not found")
        endif()
    endforeach()
endif()

# Build tests if enabled
if(BUILD_TESTS)
    enable_testing()
    
    file(GLOB TEST_SOURCES "tests/*.cpp")
    foreach(test_file IN LISTS TEST_SOURCES)
        get_filename_component(test_name ${test_file} NAME_WE)
        add_executable(${test_name} ${test_file})
        configure_target(${test_name})
        target_link_libraries(${test_name} PRIVATE mylib)
        add_test(NAME ${test_name} COMMAND ${test_name})
    endforeach()
endif()

# Print configuration summary
message(STATUS "=== Configuration Summary ===")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "Compiler: ${CMAKE_CXX_COMPILER_ID}")
message(STATUS "Build tests: ${BUILD_TESTS}")
message(STATUS "Build examples: ${BUILD_EXAMPLES}")
message(STATUS "Enable warnings: ${ENABLE_WARNINGS}")
```

## Best Practices

1. **Use functions instead of macros** when possible (better scoping)
2. **Keep conditions simple** and readable
3. **Use meaningful variable names** in loops
4. **Avoid deep nesting** - extract to functions
5. **Use `break()` and `continue()`** to simplify loop logic
6. **Comment complex conditions** for clarity
7. **Prefer target-based commands** over global loops

These control statements give you powerful flow control for complex build configurations!