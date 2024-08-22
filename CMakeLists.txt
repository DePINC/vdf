cmake_minimum_required(VERSION 3.5)

project(time_vdf)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

enable_language(C CXX ASM)

option(BUILD_TEST "Build test" OFF)

if (${CMAKE_SYSTEM_NAME} MATCHES Darwin)
    add_compile_definitions(CHIAOSX=1)
endif()
add_compile_definitions(
    VDF_MODE=0
    FAST_MACHINE=1
)

if (${CMAKE_CXX_COMPILER_ID} MATCHES ^.*Clang.*$)
    set(PIEFLAG "-fno-PIE")
else()
    set(PIEFLAG "-no-pie")
    set(PIELINKOPTS "-no-pie")
endif()

# LTO flag will only apply to release version
if (CMAKE_BUILD_TYPE MATCHES Debug)
    add_compile_definitions(DEBUG=1)
    set(OPTIMIZEFLAGS "-O0")
else()
    add_compile_definitions(NDEBUG=1)
    set(LTOFLAG "-flto")
    message(STATUS "LTO is on")
    set(OPTIMIZEFLAGS "-O2")
endif()
set(VISIBILITY_HIDDEN_FLAG "-fvisibility=hidden")

include(FetchContent)
FetchContent_Declare(
    chiavdf
    GIT_REPOSITORY https://github.com/Chia-Network/chiavdf
    GIT_TAG 1.0.7
    PATCH_COMMAND git apply --reject ${CMAKE_CURRENT_SOURCE_DIR}/chiavdf-improved.patch || true
)
FetchContent_MakeAvailable(chiavdf)

set(UINT128_SRC ${chiavdf_SOURCE_DIR}/src/uint128_t/uint128_t.cpp)

# Setup the way to retrieve GMP source
if (WIN32)
    set(STATIC_CXXFLAGS "-static -static-libstdc++ -static-libgcc")
endif()

message(STATUS "flags: ${STATIC_CXXFLAGS}")
if(STATIC_CXXFLAGS)
    string(REPLACE " " ";" TARGET_STATIC_CXXFLAGS ${STATIC_CXXFLAGS})
endif(STATIC_CXXFLAGS)

set(GMP_PREFIX_DIR ${CMAKE_CURRENT_BINARY_DIR}/gmp)

if (APPLE)
    set(GMP_URL https://gmplib.org/download/gmp/gmp-6.3.0.tar.xz)
    set(GMP_HASH SHA256=a3c2b80201b89e68616f4ad30bc66aee4927c3ce50e33929ca819d5c43538898)
    set(GMP_INST_CMD make install)
    set(GMP_CONFIG_CMD ./configure CC=gcc --enable-cxx --enable-static --disable-shared --with-pic --prefix=${GMP_PREFIX_DIR})
    set(GMP_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmp.a)
    set(GMPXX_LIBRARIES ${GMP_PREFIX_DIR}/lib/libgmpxx.a)
else()
    set(GMP_URL https://github.com/wbhart/mpir/archive/refs/tags/mpir-3.0.0.tar.gz)
    set(GMP_HASH SHA256=86a5039badc3e6738219a262873a1db5513405e15ece9527b718fcd0fac09bb2)
    set(GMP_INST_CMD make install && ln -f -s ${GMP_PREFIX_DIR}/include/mpir.h ${GMP_PREFIX_DIR}/include/gmp.h && ln -f -s ${GMP_PREFIX_DIR}/include/mpirxx.h ${GMP_PREFIX_DIR}/include/gmpxx.h)
    set(GMP_CONFIG_CMD ./autogen.sh && ./configure --enable-cxx --enable-static --disable-shared --prefix=${GMP_PREFIX_DIR} --host=${TOOLCHAIN_PREFIX} CC_FOR_BUILD=gcc CXXFLAGS=${STATIC_CXXFLAGS})
    set(GMP_LIBRARIES ${GMP_PREFIX_DIR}/lib/libmpir.a)
    set(GMPXX_LIBRARIES ${GMP_PREFIX_DIR}/lib/libmpirxx.a)
endif()

set(GMP_SOURCE_DIR ${GMP_PREFIX_DIR}/src)
include(ExternalProject)
ExternalProject_Add(GMP
    URL ${GMP_URL}
    URL_HASH ${GMP_HASH}
    SOURCE_DIR "${GMP_SOURCE_DIR}"
    CONFIGURE_COMMAND ${GMP_CONFIG_CMD}
    TEST_COMMAND ""
    BUILD_IN_SOURCE 1
    DOWNLOAD_EXTRACT_TIMESTAMP true
    INSTALL_COMMAND ${GMP_INST_CMD}
)
set(GMP_INCLUDE_DIR ${GMP_PREFIX_DIR}/include)
set(GMPXX_INCLUDE_DIR ${GMP_PREFIX_DIR}/include)

set(CHIAVDF_SRC
    ${chiavdf_SOURCE_DIR}/src/refcode/lzcnt.c
)
set(VDF_LIB_SRCS
    src/vdf_computer.cpp
    src/vdf_utils.cpp
    ${CHIAVDF_SRC}
)
add_library(time_vdf STATIC ${VDF_LIB_SRCS} ${CHIAPOS_LIB_SRCS})
add_dependencies(time_vdf GMP)
target_compile_options(time_vdf PUBLIC ${PIEFLAG} ${OPTIMIZEFLAGS} ${VISIBILITY_HIDDEN_FLAG})
target_compile_definitions(time_vdf PUBLIC ${CHIAPOS_COMPILE_DEFS})
target_compile_features(time_vdf PUBLIC cxx_std_17)
target_link_options(time_vdf PUBLIC ${PIELINKOPTS} ${LTOFLAG})
target_include_directories(time_vdf PUBLIC
    ${GMP_INCLUDE_DIR}
    ${GMPXX_INCLUDE_DIR}
    ${chiavdf_SOURCE_DIR}/src
)
target_link_libraries(time_vdf PUBLIC ${GMP_LIBRARIES} ${GMPXX_LIBRARIES})

set(INSTALL_INCLUDE_FILES
    src/vdf_types.h
    src/vdf_utils.h
    src/vdf_computer.h
    ${chiavdf_SOURCE_DIR}/src/verifier.h
)
install(FILES ${INSTALL_INCLUDE_FILES} DESTINATION include)
install(TARGETS time_vdf DESTINATION lib)

if (BUILD_TEST)
    set(BUILD_GMOCK OFF CACHE INTERNAL "Do not compile GMOCK")
    set(INSTALL_GTEST OFF CACHE INTERNAL "Do not install GTEST")
    FetchContent_Declare(
        gtest
        GIT_REPOSITORY https://github.com/google/googletest
        GIT_TAG v1.13.0
    )
    FetchContent_MakeAvailable(gtest)
    enable_testing()
    set(VDF_TEST_SRCS
        src/vdf_test.cpp
        src/vdf_utils.cpp
        src/vdf_computer.cpp
    )
    add_executable(vdf_test ${VDF_TEST_SRCS})
    add_dependencies(vdf_test GMP)
    target_link_options(vdf_test PRIVATE ${TARGET_STATIC_CXXFLAGS})
    target_link_libraries(vdf_test PRIVATE GTest::gtest_main time_vdf)
    add_test(vdf_test ${CMAKE_CURRENT_BINARY_DIR}/vdf_test)
    if (INCLUDE_CHIAPOS)
        add_executable(chiapos_test src/simple_test.cpp src/vdf_utils.cpp)
        target_link_libraries(chiapos_test PRIVATE GTest::gtest GTest::gtest_main time_vdf blake3)
        target_include_directories(chiapos_test PRIVATE
            ${chiapos_SOURCE_DIR}/src
            ${chiapos_SOURCE_DIR}/lib/include
            ${BLAKE3_INCLUDE_DIR}
        )
    endif()
endif(BUILD_TEST)