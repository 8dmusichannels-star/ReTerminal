cmake_minimum_required(VERSION 3.16)
project(proot C)

# ---------------------------------------------------------------------------
# Paths / general setup
# ---------------------------------------------------------------------------
set(SRC ${CMAKE_CURRENT_SOURCE_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

find_program(GIT_EXECUTABLE git REQUIRED)
find_program(PYTHON_EXECUTABLE python3)
find_program(SWIG_EXECUTABLE swig)
find_program(OBJCOPY_EXECUTABLE objcopy REQUIRED)
find_program(OBJDUMP_EXECUTABLE objdump REQUIRED)
find_program(STRIP_EXECUTABLE strip REQUIRED)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED OFF)

# ---------------------------------------------------------------------------
# talloc: build directly from vendored source (no pkg-config, no subdirectory
# CMakeLists.txt) since pkg-config can't find a cross-compiled talloc.pc
# when building for Android.
# ---------------------------------------------------------------------------
add_library(talloc STATIC
    ${SRC}/talloc/talloc.c
    ${SRC}/talloc/replace.c
    ${SRC}/talloc/closefrom.c
)
target_include_directories(talloc PUBLIC ${SRC}/talloc)
target_compile_definitions(talloc PRIVATE HAVE_CONFIG_H)

set(TALLOC_INCLUDE_DIRS ${SRC}/talloc)
set(TALLOC_LIBRARIES talloc)
set(TALLOC_CFLAGS_OTHER "")

include(FindPkgConfig)
pkg_check_modules(LIBARCHIVE libarchive)   # only needed for the optional 'care' target

# ---------------------------------------------------------------------------
# Common compiler / preprocessor flags (equivalent of CPPFLAGS/CFLAGS/LDFLAGS)
# ---------------------------------------------------------------------------
add_compile_definitions(_FILE_OFFSET_BITS=64 _GNU_SOURCE)
include_directories(
    ${CMAKE_BINARY_DIR}
    ${SRC}
    ${SRC}/../lib/uthash/include
    ${TALLOC_INCLUDE_DIRS}
)

add_compile_options(-g -Wall -Wextra -O2 ${TALLOC_CFLAGS_OTHER})

# -Wl,-z,noexecstack for every link
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-z,noexecstack")
link_libraries(${TALLOC_LIBRARIES})

# ---------------------------------------------------------------------------
# Version string (GIT_VERSION-GIT_COMMIT), used by build.h
# ---------------------------------------------------------------------------
execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --tags
    COMMAND_ERROR_IS_FATAL NONE
    WORKING_DIRECTORY ${SRC}
    OUTPUT_VARIABLE GIT_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-list --all --max-count=1
    WORKING_DIRECTORY ${SRC}
    OUTPUT_VARIABLE GIT_COMMIT_FULL
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
string(SUBSTRING "${GIT_COMMIT_FULL}" 0 8 GIT_COMMIT)
set(PROOT_VERSION "${GIT_VERSION}-${GIT_COMMIT}")

# ---------------------------------------------------------------------------
# Optional python extension (needs swig + python*-config --embed)
# ---------------------------------------------------------------------------
set(HAVE_PYTHON_EXTENSION FALSE)
if(SWIG_EXECUTABLE AND PYTHON_EXECUTABLE)
    execute_process(
        COMMAND ${PYTHON_EXECUTABLE}-config --includes
        OUTPUT_VARIABLE PYTHON_CONFIG_INCLUDES
        OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE PYTHON_CONFIG_RESULT
    )
    if(PYTHON_CONFIG_RESULT EQUAL 0)
        set(HAVE_PYTHON_EXTENSION TRUE)
        execute_process(
            COMMAND ${PYTHON_EXECUTABLE} -c "import sys; print(sys.version_info.major)"
            OUTPUT_VARIABLE PYTHON_MAJOR_VERSION
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        execute_process(
            COMMAND ${PYTHON_EXECUTABLE} -c "import sys; print('--embed' if sys.hexversion > 0x03080000 else '')"
            OUTPUT_VARIABLE PYTHON_EMBED
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        execute_process(
            COMMAND ${PYTHON_EXECUTABLE}-config --ldflags ${PYTHON_EMBED}
            OUTPUT_VARIABLE PYTHON_CONFIG_LDFLAGS
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
    endif()
endif()

# ---------------------------------------------------------------------------
# Feature checks (process_vm, seccomp_filter) -> HAVE_PROCESS_VM / HAVE_SECCOMP_FILTER
# ---------------------------------------------------------------------------
include(CheckCSourceCompiles)

check_c_source_compiles("
#include <sys/uio.h>
int main(void) {
    struct iovec local, remote;
    process_vm_readv(0, &local, 1, &remote, 1, 0);
    return 0;
}" HAVE_PROCESS_VM)

check_c_source_compiles("
#include <linux/seccomp.h>
#ifndef SECCOMP_MODE_FILTER
#error no seccomp filter
#endif
int main(void) { return 0; }" HAVE_SECCOMP_FILTER)

# ---------------------------------------------------------------------------
# build.h (auto-generated header, equivalent of the Makefile's build.h rule)
# ---------------------------------------------------------------------------
set(BUILD_H ${CMAKE_BINARY_DIR}/build.h)
file(WRITE ${BUILD_H} "/* This file is auto-generated, edit at your own risk.  */\n")
file(APPEND ${BUILD_H} "#ifndef BUILD_H\n#define BUILD_H\n")
if(PROOT_VERSION)
    file(APPEND ${BUILD_H} "#undef VERSION\n#define VERSION \"${PROOT_VERSION}\"\n")
endif()
if(HAVE_PYTHON_EXTENSION)
    file(APPEND ${BUILD_H} "#define HAVE_PYTHON_EXTENSION\n")
endif()
if(HAVE_PROCESS_VM)
    file(APPEND ${BUILD_H} "#define HAVE_PROCESS_VM\n")
endif()
if(HAVE_SECCOMP_FILTER)
    file(APPEND ${BUILD_H} "#define HAVE_SECCOMP_FILTER\n")
endif()
file(APPEND ${BUILD_H} "#endif /* BUILD_H */\n")

# ---------------------------------------------------------------------------
# Core object sources
# ---------------------------------------------------------------------------
set(PROOT_SOURCES
    cli/cli.c
    cli/proot.c
    cli/note.c
    execve/enter.c
    execve/exit.c
    execve/shebang.c
    execve/elf.c
    execve/ldso.c
    execve/auxv.c
    execve/aoxp.c
    path/binding.c
    path/glue.c
    path/canon.c
    path/path.c
    path/proc.c
    path/temp.c
    syscall/seccomp.c
    syscall/syscall.c
    syscall/chain.c
    syscall/enter.c
    syscall/exit.c
    syscall/sysnum.c
    syscall/socket.c
    syscall/heap.c
    syscall/rlimit.c
    tracee/tracee.c
    tracee/mem.c
    tracee/reg.c
    tracee/event.c
    ptrace/ptrace.c
    ptrace/user.c
    ptrace/wait.c
    extension/extension.c
    extension/kompat/kompat.c
    extension/fake_id0/fake_id0.c
    extension/link2symlink/link2symlink.c
    extension/portmap/portmap.c
    extension/portmap/map.c
)

if(HAVE_PYTHON_EXTENSION)
    list(APPEND PROOT_SOURCES
        extension/python/python.c
        extension/python/python_extension.c
    )
endif()

set(CARE_SOURCES
    cli/care.c
    extension/care/care.c
    extension/care/final.c
    extension/care/extract.c
    extension/care/archive.c
)

# All hand-written sources depend on build.h being generated first.
set_source_files_properties(${PROOT_SOURCES} ${CARE_SOURCES}
    PROPERTIES OBJECT_DEPENDS ${BUILD_H})

# ---------------------------------------------------------------------------
# Loader(s): a tiny freestanding static binary, later objcopy'd into a
# regular .o and linked into proot/care as embedded data (see loader/loader.c).
# Built per-arch: native, and 32-bit if the compiler can produce it (mirrors
# HAS_LOADER_32BIT check against arch.h in the original Makefile).
# ---------------------------------------------------------------------------
function(build_proot_loader suffix extra_cflags)
    set(loader_dir ${CMAKE_BINARY_DIR}/loader${suffix})
    file(MAKE_DIRECTORY ${loader_dir})

    set(loader_elf ${loader_dir}/loader${suffix})
    set(loader_stripped ${loader_dir}/loader${suffix}.elf)
    set(loader_obj ${CMAKE_BINARY_DIR}/loader${suffix}-wrapped.o)

    add_custom_command(
        OUTPUT ${loader_elf}
        COMMAND ${CMAKE_C_COMPILER} ${extra_cflags} -fPIC -ffreestanding
                -static -nostdlib -Wl,-Ttext=0x0,-z,noexecstack
                -o ${loader_elf}
                ${SRC}/loader/loader.c ${SRC}/loader/assembly.S
        DEPENDS ${SRC}/loader/loader.c ${SRC}/loader/assembly.S ${SRC}/arch.h
        COMMENT "Building freestanding loader${suffix}"
        VERBATIM
    )

    add_custom_command(
        OUTPUT ${loader_stripped}
        COMMAND ${CMAKE_COMMAND} -E copy ${loader_elf} ${loader_stripped}
        COMMAND ${STRIP_EXECUTABLE} ${loader_stripped}
        DEPENDS ${loader_elf}
        VERBATIM
    )

    add_custom_command(
        OUTPUT ${loader_obj}
        COMMAND ${OBJCOPY_EXECUTABLE}
                --input-target binary
                --output-target elf64-x86-64
                --binary-architecture i386:x86-64
                ${loader_stripped} ${loader_obj}
        WORKING_DIRECTORY ${loader_dir}
        DEPENDS ${loader_stripped}
        COMMENT "Embedding loader${suffix} into object file"
        VERBATIM
    )

    set(LOADER_OBJ${suffix} ${loader_obj} PARENT_SCOPE)
endfunction()

# NOTE: the --output-target/--binary-architecture pair should match the
# host's actual object format; adjust if not targeting x86-64
# (this mirrors the original Makefile's use of `objdump -f cli/cli.o`).
build_proot_loader("" "")
set(LOADER_OBJECTS ${LOADER_OBJ})

# 32-bit loader is only built if the toolchain supports -m32 (equivalent of
# checking HAS_LOADER_32BIT from arch.h); enable manually if needed:
option(BUILD_LOADER_32BIT "Also build the 32-bit loader" OFF)
if(BUILD_LOADER_32BIT)
    build_proot_loader("-m32" "-m32")
    list(APPEND LOADER_OBJECTS ${LOADER_OBJ-m32})
endif()

# ---------------------------------------------------------------------------
# proot
# ---------------------------------------------------------------------------
add_executable(proot ${PROOT_SOURCES} ${LOADER_OBJECTS})
if(HAVE_PYTHON_EXTENSION)
    target_include_directories(proot PRIVATE ${PYTHON_CONFIG_INCLUDES})
    target_link_options(proot PRIVATE ${PYTHON_CONFIG_LDFLAGS})
endif()

# ---------------------------------------------------------------------------
# care (optional; requires libarchive)
# ---------------------------------------------------------------------------
if(LIBARCHIVE_FOUND)
    add_executable(care ${PROOT_SOURCES} ${CARE_SOURCES} ${LOADER_OBJECTS})
    target_link_libraries(care PRIVATE ${LIBARCHIVE_LIBRARIES})
    target_include_directories(care PRIVATE ${LIBARCHIVE_INCLUDE_DIRS})
else()
    message(STATUS "libarchive not found: 'care' target disabled")
endif()

# ---------------------------------------------------------------------------
# Install
# ---------------------------------------------------------------------------
install(TARGETS proot RUNTIME DESTINATION bin)
if(TARGET care)
    install(TARGETS care RUNTIME DESTINATION bin)
endif()
