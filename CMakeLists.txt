cmake_minimum_required(VERSION 3.2)

project(chiavdf)

enable_language(C CXX ASM)

execute_process(COMMAND uname OUTPUT_VARIABLE UNAME)

find_package(Threads REQUIRED)

set(LZCNT src/refcode/lzcnt.c)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    set(NOPIE "-fno-PIE")
else()
    set(NOPIE "-no-pie")
endif()

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(OPT_CFLAGS "-O0")
else()
    set(OPT_CFLAGS "-O2")
endif()

# Setup the way to retrieve GMP source
include(ExternalProject)

set(GMP_PREFIX_DIR ${CMAKE_BINARY_DIR}/gmp)
set(GMP_SOURCE_DIR ${GMP_PREFIX_DIR}/src)
ExternalProject_Add(GMP
    URL https://gmplib.org/download/gmp/gmp-6.2.1.tar.lz
    URL_HASH SHA256=2c7f4f0d370801b2849c48c9ef3f59553b5f1d3791d070cffb04599f9fc67b41
    SOURCE_DIR "${GMP_SOURCE_DIR}"
    CONFIGURE_COMMAND ./configure --enable-cxx --enable-fat --prefix=${GMP_PREFIX_DIR} --host=${TOOLCHAIN_PREFIX} CC_FOR_BUILD=gcc
    TEST_COMMAND ""
    BUILD_IN_SOURCE 1
    PATCH_COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/src/lib/gmp-patch-6.2.1/compat.c ${GMP_SOURCE_DIR}/compat.c && cp ${CMAKE_CURRENT_SOURCE_DIR}/src/lib/gmp-patch-6.2.1/longlong.h ${GMP_SOURCE_DIR}/longlong.h && cp ${CMAKE_CURRENT_SOURCE_DIR}/src/lib/gmp-patch-6.2.1/mpz/inp_raw.c ${GMP_SOURCE_DIR}/mpz/inp_raw.c
    )
set(GMP_INCLUDE_DIR ${GMP_PREFIX_DIR}/include)
set(GMPXX_INCLUDE_DIR ${GMP_PREFIX_DIR}/include)

if (WIN32)
    set(GMP_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmp.a)
    set(GMPXX_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmpxx.a)
else()
    set(GMP_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmp.so)
    set(GMPXX_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmpxx.so)
endif()

# Setup ASIO
set(ASIO_PREFIX_DIR ${CMAKE_BINARY_DIR}/asio)
set(ASIO_SOURCE_DIR ${ASIO_PREFIX_DIR}/src)
ExternalProject_Add(ASIO
    URL https://master.dl.sourceforge.net/project/asio/asio/1.22.2%20%28Stable%29/asio-1.22.2.tar.bz2
    URL_HASH SHA256=776bb781eee8022a3eed71de28f05bf8ba09741db2e57ad8cc420aa0884cbc6c
    SOURCE_DIR ${ASIO_SOURCE_DIR}
    CONFIGURE_COMMAND ./configure --prefix=${ASIO_PREFIX_DIR}
    TEST_COMMAND ""
    BUILD_IN_SOURCE 1
    )

include_directories(
    ${GMP_INCLUDE_DIR}
    ${GMPXX_INCLUDE_DIR}
    ${ASIO_PREFIX_DIR}/include
    src
    )

# verifier_test
add_executable(verifier_test
    src/verifier_test.cpp
    ${LZCNT}
)
target_link_libraries(verifier_test PRIVATE ${GMP_LIBRARIES} ${GMPXX_LIBRARIES} Threads::Threads)

# compile_asm
if (WIN32)
    set(UINT128_SRCS src/uint128_t/uint128_t.cpp)
    set(COMPILE_ASM_EXE ${CMAKE_BINARY_DIR}/compile_asm.exe)
else()
    set(COMPILE_ASM_EXE ${CMAKE_BINARY_DIR}/compile_asm)
endif()
set(COMPILE_ASM_SRC src/compile_asm.cpp ${UINT128_SRCS})
add_executable(compile_asm ${COMPILE_ASM_SRC})
add_dependencies(compile_asm GMP)
target_compile_options(compile_asm PRIVATE ${OPT_CFLAGS} -g)
target_link_options(compile_asm PRIVATE -flto ${NOPIE})
target_link_libraries(compile_asm PRIVATE ${GMP_LIBRARIES} ${GMPXX_LIBRARIES})

add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/asm_compiled.s COMMAND ${COMPILE_ASM_EXE} WORKING_DIRECTORY ${CMAKE_BINARY_DIR} DEPENDS compile_asm)
add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/avx2_asm_compiled.s COMMAND ${COMPILE_ASM_EXE} avx2 WORKING_DIRECTORY ${CMAKE_BINARY_DIR} DEPENDS compile_asm)
add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/avx512_asm_compiled.s COMMAND ${COMPILE_ASM_EXE} avx512 WORKING_DIRECTORY ${CMAKE_BINARY_DIR} DEPENDS compile_asm)

# targets require asm files
function(compile_target TARGET_EXE_NAME)
    add_executable(${TARGET_EXE_NAME} src/${TARGET_EXE_NAME}.cpp ${LZCNT} ${CMAKE_BINARY_DIR}/asm_compiled.s ${CMAKE_BINARY_DIR}/avx2_asm_compiled.s ${CMAKE_BINARY_DIR}/avx512_asm_compiled.s ${UINT128_SRCS})
    add_dependencies(${TARGET_EXE_NAME} ASIO)
    target_compile_features(${TARGET_EXE_NAME} PRIVATE cxx_std_17)
    target_compile_definitions(${TARGET_EXE_NAME} PRIVATE VDF_MODE=0 FAST_MACHINE=1)
    if (UNAME STREQUAL "Darwin")
        target_compile_definitions(${TARGET_EXE_NAME} PRIVATE CHIAOSX=1)
    endif()
    target_compile_options(${TARGET_EXE_NAME} PRIVATE -pthread ${NOPIE} -fvisibility=hidden ${OPT_CFLAGS} -g)
    target_link_options(${TARGET_EXE_NAME} PRIVATE -flto ${NOPIE})
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        target_compile_options(${TARGET_EXE_NAME} PRIVATE -flto)
        target_link_options(${TARGET_EXE_NAME} PRIVATE -flto)
    endif()
    target_link_libraries(${TARGET_EXE_NAME} PRIVATE ${GMP_LIBRARIES} ${GMPXX_LIBRARIES} Threads::Threads)
    if (WIN32)
        target_link_libraries(${TARGET_EXE_NAME} PRIVATE ws2_32)
    endif()
endfunction()

compile_target(vdf_client)
compile_target(vdf_bench)
compile_target(prover_test)
compile_target(1weso_test)
compile_target(2weso_test)
compile_target(avx512_test)
