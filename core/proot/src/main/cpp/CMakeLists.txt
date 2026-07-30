cmake_minimum_required(VERSION 3.13)
project(proot C ASM)

# ----------------------------------------------------------------------
# Basic settings
# ----------------------------------------------------------------------

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

option(PROOT_WITH_LIBANDROID_SHMEM "Enable libandroid-shmem support" OFF)
option(PROOT_UNBUNDLE_LOADER_PATH "Target directory to install the loader separately (empty = use bundled loader)" "")

# This build targets arm64-v8a only. No cross-ABI / multi-triple support:
# a single target triple is used for both the main build and the loader.
set(MAIN_TARGET_TRIPLE "aarch64-none-linux-android26")

if(NOT CMAKE_C_COMPILER)
    set(CMAKE_C_COMPILER "gcc")
endif()

find_program(OBJCOPY_EXECUTABLE NAMES objcopy)
find_program(OBJDUMP_EXECUTABLE NAMES objdump)
find_program(STRIP_EXECUTABLE   NAMES strip)
find_program(GIT_EXECUTABLE     NAMES git)
find_program(READELF_EXECUTABLE NAMES readelf)

set(SRC ${CMAKE_CURRENT_SOURCE_DIR})

# ----------------------------------------------------------------------
# Compiler flags
# ----------------------------------------------------------------------

add_compile_definitions(_FILE_OFFSET_BITS=64 _GNU_SOURCE)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${SRC})

add_compile_options(-Wall -Wextra -O2)

if(PROOT_WITH_LIBANDROID_SHMEM)
    add_compile_definitions(WITH_LIBANDROID_SHMEM)
endif()

# ----------------------------------------------------------------------
# Feature detection via arch.h (HAS_LOADER_32BIT / HAS_POKEDATA_WORKAROUND
# / LOADER_ADDRESS / LOADER_ARCH_CFLAGS), resolved for the single
# arm64-v8a target triple.
# ----------------------------------------------------------------------

function(define_from_arch_h out_var macro_name target_triple)
    if(target_triple)
        set(_target_flag --target=${target_triple})
    else()
        set(_target_flag "")
    endif()
    execute_process(
        COMMAND ${CMAKE_C_COMPILER} ${_target_flag} -E -dM -DNO_LIBC_HEADER ${SRC}/arch.h
        OUTPUT_VARIABLE _arch_defs
        ERROR_QUIET
    )
    string(REGEX MATCH "#define[ \t]+${macro_name}[ \t]+([^\n]*)" _match "${_arch_defs}")
    if(_match)
        set(${out_var} "${CMAKE_MATCH_1}" PARENT_SCOPE)
    else()
        set(${out_var} "" PARENT_SCOPE)
    endif()
endfunction()

define_from_arch_h(HAS_LOADER_32BIT "HAS_LOADER_32BIT" ${MAIN_TARGET_TRIPLE})
define_from_arch_h(HAS_POKEDATA_WORKAROUND "HAS_POKEDATA_WORKAROUND" ${MAIN_TARGET_TRIPLE})

# ----------------------------------------------------------------------
# Source files
# ----------------------------------------------------------------------

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
    path/f2fs-bug.c
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
    syscall/pipe_shadow.c
    tracee/tracee.c
    tracee/mem.c
    tracee/reg.c
    tracee/event.c
    tracee/seccomp.c
    tracee/statx.c
    ptrace/ptrace.c
    ptrace/user.c
    ptrace/wait.c
    extension/extension.c
    extension/ashmem_memfd/ashmem_memfd.c
    extension/kompat/kompat.c
    extension/fake_id0/chown.c
    extension/fake_id0/chroot.c
    extension/fake_id0/getsockopt.c
    extension/fake_id0/sendmsg.c
    extension/fake_id0/socket.c
    extension/fake_id0/open.c
    extension/fake_id0/unlink.c
    extension/fake_id0/rename.c
    extension/fake_id0/chmod.c
    extension/fake_id0/utimensat.c
    extension/fake_id0/access.c
    extension/fake_id0/exec.c
    extension/fake_id0/link.c
    extension/fake_id0/symlink.c
    extension/fake_id0/mk.c
    extension/fake_id0/stat.c
    extension/fake_id0/helper_functions.c
    extension/fake_id0/fake_id0.c
    extension/hidden_files/hidden_files.c
    extension/mountinfo/mountinfo.c
    extension/port_switch/port_switch.c
    extension/sysvipc/sysvipc.c
    extension/sysvipc/sysvipc_msg.c
    extension/sysvipc/sysvipc_sem.c
    extension/sysvipc/sysvipc_shm.c
    extension/link2symlink/link2symlink.c
    extension/fix_symlink_size/fix_symlink_size.c
)

list(TRANSFORM PROOT_SOURCES PREPEND "${SRC}/")

# ----------------------------------------------------------------------
# build.h generation (VERSION + HAVE_* feature macros)
# ----------------------------------------------------------------------

set(BUILD_H "${CMAKE_CURRENT_BINARY_DIR}/build.h")

include(CheckCSourceCompiles)

check_c_source_compiles("
#include <sys/uio.h>
int main(void){ struct iovec local, remote; process_vm_readv(0,&local,1,&remote,1,0); return 0; }
" HAVE_PROCESS_VM)

check_c_source_compiles("
#include <linux/seccomp.h>
#include <linux/filter.h>
int main(void){ return SECCOMP_MODE_FILTER; }
" HAVE_SECCOMP_FILTER)

if(GIT_EXECUTABLE)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags --dirty --abbrev=8 --always
        WORKING_DIRECTORY ${SRC}
        OUTPUT_VARIABLE GIT_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
endif()

file(WRITE ${BUILD_H} "/* This file is auto-generated, edit at your own risk.  */\n")
file(APPEND ${BUILD_H} "#ifndef BUILD_H\n#define BUILD_H\n")
if(GIT_VERSION)
    file(APPEND ${BUILD_H} "#undef VERSION\n#define VERSION \"${GIT_VERSION}\"\n")
endif()
if(HAVE_PROCESS_VM)
    file(APPEND ${BUILD_H} "#define HAVE_PROCESS_VM\n")
endif()
if(HAVE_SECCOMP_FILTER)
    file(APPEND ${BUILD_H} "#define HAVE_SECCOMP_FILTER\n")
endif()
file(APPEND ${BUILD_H} "#endif /* BUILD_H */\n")

# ----------------------------------------------------------------------
# Bundled talloc library (built from source instead of relying on a
# system -ltalloc, which is not available on Android/NDK sysroots)
# ----------------------------------------------------------------------

add_library(talloc STATIC
    ${SRC}/talloc/talloc.c
    ${SRC}/talloc/replace.c
    ${SRC}/talloc/closefrom.c
)
target_compile_definitions(talloc PRIVATE HAVE_CONFIG_H __STDC_WANT_LIB_EXT1__=1)
target_include_directories(talloc PRIVATE ${SRC}/talloc)
target_include_directories(talloc INTERFACE ${SRC}/talloc)
target_compile_options(talloc PRIVATE -w)

# ----------------------------------------------------------------------
# Linker flags (build-id / rosegment feature tests)
# ----------------------------------------------------------------------

set(LDFLAGS_COMMON -Wl,-z,noexecstack)
if(PROOT_WITH_LIBANDROID_SHMEM)
    list(APPEND LDFLAGS_COMMON -landroid-shmem)
endif()

execute_process(
    COMMAND ${CMAKE_C_COMPILER} -xc - -Wl,--build-id=none -o /dev/null
    INPUT_FILE /dev/null
    RESULT_VARIABLE _build_id_none_result
    OUTPUT_QUIET ERROR_QUIET
)
if(_build_id_none_result EQUAL 0)
    set(BUILD_ID_NONE ",--build-id=none")
endif()

execute_process(
    COMMAND ${CMAKE_C_COMPILER} -xc - -Wl,--rosegment -o /dev/null
    INPUT_FILE /dev/null
    RESULT_VARIABLE _rosegment_result
    OUTPUT_QUIET ERROR_QUIET
)
if(_rosegment_result EQUAL 0)
    set(ROSEGMENT ",--rosegment")
endif()

# ----------------------------------------------------------------------
# Loader (loader.c + assembly.S -> embedded binary object)
# Single arm64-v8a triple only, no companion 32-bit loader, no per-ABI
# branching.
# ----------------------------------------------------------------------

function(build_loader)
    set(loader_name "loader")
    set(loader_dir "${CMAKE_CURRENT_BINARY_DIR}/loader")
    file(MAKE_DIRECTORY ${loader_dir})

    define_from_arch_h(LOADER_ADDRESS_VAL "LOADER_ADDRESS" ${MAIN_TARGET_TRIPLE})
    define_from_arch_h(LOADER_ARCH_CFLAGS_VAL "LOADER_ARCH_CFLAGS" ${MAIN_TARGET_TRIPLE})

    if(NOT LOADER_ADDRESS_VAL)
        message(FATAL_ERROR "build_loader: could not resolve LOADER_ADDRESS for triple '${MAIN_TARGET_TRIPLE}' (arch.h missing a branch for this target?)")
    endif()

    add_library(${loader_name}_objs OBJECT
        ${SRC}/loader/loader.c
        ${SRC}/loader/assembly.S
    )
    target_compile_options(${loader_name}_objs PRIVATE
        --target=${MAIN_TARGET_TRIPLE} -fPIC -ffreestanding ${LOADER_ARCH_CFLAGS_VAL}
    )

    add_executable(${loader_name} $<TARGET_OBJECTS:${loader_name}_objs>)
    target_link_options(${loader_name} PRIVATE
        --target=${MAIN_TARGET_TRIPLE} -static -nostdlib
        -Wl${BUILD_ID_NONE},-Ttext=${LOADER_ADDRESS_VAL}${ROSEGMENT},-z,noexecstack
    )
    set_target_properties(${loader_name} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${loader_dir}
    )

    # loader binary -> stripped copy -> embed into an object file via objcopy
    set(loader_stripped "${loader_dir}/${loader_name}.stripped")
    add_custom_command(
        OUTPUT ${loader_stripped}
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${loader_name}> ${loader_stripped}
        COMMAND ${STRIP_EXECUTABLE} ${loader_stripped}
        DEPENDS ${loader_name}
        VERBATIM
    )

    set(loader_wrapped_obj "${loader_dir}/${loader_name}-wrapped.o")
    add_custom_command(
        OUTPUT ${loader_wrapped_obj}
        COMMAND ${OBJCOPY_EXECUTABLE}
                --input-target=binary
                --output-target=elf64-littleaarch64
                --binary-architecture aarch64
                ${loader_stripped} ${loader_wrapped_obj}
        DEPENDS ${loader_stripped}
        COMMENT "GEN ${loader_name}-wrapped.o (embedded loader binary, ${MAIN_TARGET_TRIPLE})"
        VERBATIM
    )

    set(loader_WRAPPED_OBJ ${loader_wrapped_obj} PARENT_SCOPE)
endfunction()

if(NOT PROOT_UNBUNDLE_LOADER_PATH)
    build_loader()
    list(APPEND PROOT_EXTRA_OBJECTS ${loader_WRAPPED_OBJ})
endif()

if(HAS_POKEDATA_WORKAROUND AND READELF_EXECUTABLE)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/loader/loader-info.c
        COMMAND ${READELF_EXECUTABLE} -s $<TARGET_FILE:loader> |
                awk -f ${SRC}/loader/loader-info.awk > ${CMAKE_CURRENT_BINARY_DIR}/loader/loader-info.c
        DEPENDS loader
        VERBATIM
    )
    list(APPEND PROOT_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/loader/loader-info.c)
endif()

# ----------------------------------------------------------------------
# Main target: proot
# ----------------------------------------------------------------------

add_executable(proot ${PROOT_SOURCES} ${PROOT_EXTRA_OBJECTS})
target_link_libraries(proot PRIVATE talloc ${LDFLAGS_COMMON})

if(PROOT_UNBUNDLE_LOADER_PATH)
    target_compile_definitions(proot PRIVATE PROOT_UNBUNDLE_LOADER="${PROOT_UNBUNDLE_LOADER_PATH}")
endif()

# Package the proot executable as libproot.so so Android's packaging
# system (jniLibs) picks it up and extracts it with the correct
# permissions at install time. It is still a normal ELF executable,
# only the name/prefix/suffix mimic a shared library for packaging
# purposes.
set_target_properties(proot PROPERTIES
    OUTPUT_NAME "proot"
    PREFIX "lib"
    SUFFIX ".so"
)

add_custom_command(
    TARGET proot POST_BUILD
    COMMAND ${STRIP_EXECUTABLE} --strip-unneeded $<TARGET_FILE:proot>
    COMMENT "Stripping libproot.so"
)

# ----------------------------------------------------------------------
# Install
# ----------------------------------------------------------------------

include(GNUInstallDirs)
install(TARGETS proot LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                       RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR})

if(PROOT_UNBUNDLE_LOADER_PATH)
    install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/loader/loader
            DESTINATION ${PROOT_UNBUNDLE_LOADER_PATH}
            RENAME loader)
endif()
