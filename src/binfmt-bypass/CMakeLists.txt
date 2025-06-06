# needed for LINK_OPTIONS/target_link_options
cmake_minimum_required(VERSION 3.13)

project(appimage-binfmt-bypass)

# configure names globally
set(bypass_bin binfmt-bypass)
set(binfmt_interpreter binfmt-interpreter)
set(bypass_lib lib${bypass_bin})
set(preload_lib ${bypass_bin}-preload)
# we're not using "i386" or "armhf" but instead the more generic "32bit" suffix, making things a little easier in the
# code
set(preload_lib_32bit ${preload_lib}_32bit)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

string(TOLOWER ${CMAKE_C_COMPILER_ID} lower_cmake_c_compiler_id)

# on 64-bit systems such as x86_64 or aarch64/arm64v8, we need to provide a 32-bit preload library to allow the users
# to execute compatible 32-bit binaries, as the regular preload library cannot be preloaded due to the architecture
# being incompatible
# unfortunately, compiling on ARM is significantly harder than just adding a compiler flag, as gcc-multilib is missing
# it's likely easier with clang, but that remains to be verified
if(CMAKE_SYSTEM_PROCESSOR MATCHES x86_64)
    message(STATUS "64-bit target detected (${CMAKE_SYSTEM_PROCESSOR}), building 32-bit preload library as well")
    set(build_32bit_preload_library true)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES arm64 OR CMAKE_SYSTEM_PROCESSOR MATCHES aarch64)
    string(TOLOWER ${CMAKE_C_COMPILER_ID} lower_cmake_c_compiler_id)

    if(lower_cmake_c_compiler_id MATCHES clang)
        # clang-3.8, the default clang on xenial, doesn't like the templates in elf.cpp with the varying return types
        if(NOT ${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER 3.9.0)
            message(FATAL_ERROR "clang too old, please upgrade to, e.g., clang-8")
        endif()

        message(STATUS "64-bit target detected (${CMAKE_SYSTEM_PROCESSOR}), building 32-bit preload library as well")
        set(build_32bit_preload_library true)
    else()
        message(WARNING "64-bit target detected (${CMAKE_SYSTEM_PROCESSOR}), but compiling with ${lower_cmake_c_compiler_id}, cannot build 32-bit preload library")
    endif()
endif()

function(make_preload_lib_target target_name)
    # library to be preloaded when launching the patched runtime binary
    # we need to build with -fPIC, otherwise we can't use it with $LD_PRELOAD
    add_library(${target_name} SHARED preload.c logging.h)
    target_link_libraries(${target_name} PRIVATE dl)
    target_compile_options(${target_name}
        PRIVATE -fPIC
        PRIVATE -DCOMPONENT_NAME="preload"
        # hide all symbols by default
        PRIVATE -fvisibility=hidden
    )
    # compatibility with CMake < 3.13
    set_target_properties(${target_name} PROPERTIES LINK_OPTIONS
        # hide all symbols by default
        -fvisibility=hidden
    )

    # a bit of a hack, but it seems to make binfmt bypass work in really old Docker images (e.g., CentOS <= 7)
    add_custom_command(
        TARGET ${target_name}
        POST_BUILD
        COMMAND bash ${CMAKE_CURRENT_SOURCE_DIR}/fix-preload-library.sh $<TARGET_FILE_NAME:${target_name}>
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        VERBATIM
    )
endfunction()

make_preload_lib_target(${preload_lib})

if(build_32bit_preload_library)
    make_preload_lib_target(${preload_lib_32bit})

    if(CMAKE_SYSTEM_PROCESSOR MATCHES x86_64)
        if(lower_cmake_c_compiler_id MATCHES clang)
            target_compile_options(${preload_lib_32bit} PRIVATE "--target=i386-linux-gnu")
            target_link_options(${preload_lib_32bit} PRIVATE "--target=i386-linux-gnu")
        else()
            # GCC style flags
            target_compile_options(${preload_lib_32bit} PRIVATE "-m32")
            target_link_options(${preload_lib_32bit} PRIVATE "-m32")
        endif()
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES arm64 OR CMAKE_SYSTEM_PROCESSOR MATCHES aarch64)
        target_compile_options(${preload_lib_32bit} PRIVATE "--target=arm-linux-gnueabihf")
        target_link_options(${preload_lib_32bit} PRIVATE "--target=arm-linux-gnueabihf")
    else()
        message(FATAL_ERROR "unknown processor architecture: ${CMAKE_SYSTEM_PROCESSOR}")
    endif()
endif()

# we need to make the library below fully self-contained so that it works in other cgroups (e.g., in Docker containers)
# out of the box
# this allows us to check whether the path is available, and otherwise create a temporary file and use that then
# this is a workaround to existing issues using AppImages in Docker with AppImageLauncher installed on the host system
check_program(NAME xxd)

function(generate_preload_lib_header target_name)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${target_name}.h
        COMMAND xxd -i $<TARGET_FILE_NAME:${target_name}> ${target_name}.h
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        DEPENDS ${target_name}
        VERBATIM
    )
endfunction()

generate_preload_lib_header(${preload_lib})

# same story for 32-bit lib
if (build_32bit_preload_library)
    generate_preload_lib_header(${preload_lib_32bit})
endif()

# the lib provides an algorithm to extract the runtime, patch it and launch it, preloading our preload lib to make the
# AppImage think it is launched normally
# static linking is preferred, since we do not want to deal with an installed .so file, rpaths etc.
add_library(${bypass_lib} STATIC lib.cpp elf.cpp logging.h elf.h ${CMAKE_CURRENT_BINARY_DIR}/${preload_lib}.h)
target_link_libraries(${bypass_lib} PUBLIC dl)
# we need to include the preload lib headers (see below) from the binary dir
target_include_directories(${bypass_lib} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_compile_options(${bypass_lib}
    PRIVATE -DPRELOAD_LIB_NAME="$<TARGET_FILE_NAME:${preload_lib}>"
    PUBLIC -D_GNU_SOURCE
    # obviously needs to be private, otherwise the value leaks into users of this lib
    PRIVATE -DCOMPONENT_NAME="lib"
)
if(build_32bit_preload_library)
    target_compile_options(${bypass_lib}
        PRIVATE -DPRELOAD_LIB_NAME_32BIT="$<TARGET_FILE_NAME:${preload_lib_32bit}>"
    )
    target_sources(${bypass_lib} PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/${preload_lib_32bit}.h)
endif()

include(CheckCSourceCompiles)
message(STATUS "Checking whether memfd_create(...) is available")
check_c_source_compiles("
    #define _GNU_SOURCE
    #include <sys/mman.h>
    int main(int argc, char** argv) {
        memfd_create(\"test\", 0);
    }
    "
    HAVE_MEMFD_CREATE
)

if(HAVE_MEMFD_CREATE)
    target_compile_options(${bypass_lib} PUBLIC -DHAVE_MEMFD_CREATE)
else()
    message(WARNING "memfd_create not available, falling back to shm_open")
    target_link_libraries(${bypass_lib} PRIVATE rt)
endif()

add_executable(${bypass_bin} bypass_main.cpp)
target_link_libraries(${bypass_bin} ${bypass_lib})
target_compile_options(${bypass_bin}
    PRIVATE -DCOMPONENT_NAME="bin"
)

add_executable(${binfmt_interpreter} interpreter_main.cpp)
target_link_libraries(${binfmt_interpreter} ${bypass_lib})
target_compile_options(${binfmt_interpreter}
    PRIVATE -DCOMPONENT_NAME="interpreter"
    PRIVATE -DAPPIMAGELAUNCHER_PATH="${CMAKE_INSTALL_PREFIX}/${_bindir}/$<TARGET_FILE_NAME:AppImageLauncher>"
)
# static linking makes the F (fix binary) mode more reliable, as the binary as well as all its resources are completely
# preloaded by binfmt-misc, avoiding runtime dependencies on libc, libstdc++ etc.
target_link_options(${binfmt_interpreter}
    PRIVATE -static -static-libgcc -static-libstdc++
)

# no need to set rpath on bypass binary, it doesn't depend on any private libraries
# the preload lib is used via its absolute path anyway

install(
    TARGETS ${bypass_bin} ${preload_lib} ${binfmt_interpreter}
    RUNTIME DESTINATION ${_private_libdir} COMPONENT APPIMAGELAUNCHER
    LIBRARY DESTINATION ${_private_libdir} COMPONENT APPIMAGELAUNCHER
)
if(build_32bit_preload_library)
    install(
        TARGETS ${preload_lib}
        LIBRARY DESTINATION ${_private_libdir}
    )
endif()
