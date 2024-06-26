# Copyright 2020 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required (VERSION 3.2)
project (cpu_check VERSION 20181130 LANGUAGES C CXX)

# Options
# Use clang/llvm by default.
option(USE_CLANG "build with clang" ON)
# Build semi-statically by default.
option(BUILD_STATIC "build targets semi-statically linked" ON)

if (NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release)
endif(NOT CMAKE_BUILD_TYPE)

# Config header
configure_file (
	"${PROJECT_SOURCE_DIR}/config.h.in"
	"${PROJECT_BINARY_DIR}/config.h"
)
include_directories("${PROJECT_BINARY_DIR}")

set(CMAKE_C_COMPILER "/usr/bin/clang")
set(CMAKE_CXX_COMPILER "/usr/bin/clang++")

#if (USE_CLANG)
	set(CMAKE_C_COMPILER clang)
	set(CMAKE_CXX_COMPILER clang++)
	set(CC clang)
	set(CXX clang++)
#endif(USE_CLANG)

set(CMAKE_C_FLAGS_DEBUG "-g -Wall -O0")
set(CMAKE_CXX_FLAGS_DEBUG "-g -Wall -O0")
####set(CMAKE_C_FLAGS_RELEASE "-Wall -O2")
####set(CMAKE_CXX_FLAGS_RELEASE "-Wall -O2")


set(CMAKE_THREAD_LIBS_INIT "-lpthread")
set(CMAKE_HAVE_THREADS_LIBRARY 1)
set(CMAKE_USE_WIN32_THREADS_INIT 0)
set(CMAKE_USE_PTHREADS_INIT 1)
set(THREADS_PREFER_PTHREAD_FLAG ON)



set(absl_DIR ../abseil-cpp/build/install/lib/cmake/absl/)
set(ABSL_INCLUDES ../abseil-cpp/build/install/include)
include_directories("${ABSL_INCLUDES}")

set(BUILD_STATIC ON CACHE BOOL "Build statically linked binaries")

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)	# we want c11 not gnu11
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)	# we want c++17 not gnu++17

add_executable(cpu_check cpu_check.cc)
add_executable(crc32c_test crc32c_test.cc)

# TODO required for static build?
set_target_properties(cpu_check PROPERTIES LINK_SEARCH_START_STATIC ON)
set_target_properties(cpu_check PROPERTIES LINK_SEARCH_END_STATIC ON)



# Third party library - available as git submodule
add_library(farmhash third_party/farmhash/src/farmhash.cc)

add_library(avx avx.cc)
add_library(compressor compressor.cc)
add_library(crc32c crc32c.c)
add_library(crypto crypto.cc)
add_library(fvt_controller fvt_controller.cc)
add_library(hasher hasher.cc)
add_library(malign_buffer malign_buffer.cc)
add_library(pattern_generator pattern_generator.cc)
add_library(silkscreen silkscreen.cc)
add_library(utils utils.cc)


include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-march=sandybridge" ARCH_SANDYBRIDGE)
if(ARCH_SANDYBRIDGE)
	target_compile_options(farmhash PUBLIC -march=sandybridge)
	target_compile_options(crc32c PUBLIC -march=sandybridge)
endif(ARCH_SANDYBRIDGE)

if (BUILD_STATIC)
	set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
	set(CMAKE_EXE_LINKER_FLAGS "-static-libstdc++ -static-libgcc -flegacy-pass-manager -Xclang -load -Xclang /home/NEW_GIT/qed_sdc/InsertClonedInstPass.so")
        # Static linking of C/C++ standard libraries
	set(CMAKE_C_FLAGS "-flegacy-pass-manager -Xclang -load -Xclang /home/NEW_GIT/qed_sdc/InsertClonedInstPass.so" )
	set(CMAKE_CXX_FLAGS "-flegacy-pass-manager -Xclang -load -Xclang /home/NEW_GIT/qed_sdc/InsertClonedInstPass.so") 
endif(BUILD_STATIC)

file(GLOB GLOB_ABSL "../abseil-cpp/build/install/lib/*.a")

# Needs abseil
find_package(absl REQUIRED)
target_link_libraries(compressor ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(crypto  ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(fvt_controller ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(malign_buffer ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(silkscreen ${GLOB_ABSL}  /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(utils ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a)
target_link_libraries(cpu_check ${GLOB_ABSL} /usr/lib/x86_64-linux-gnu/libpthread.a -static)
#/lib/x86_64-linux-gnu/libc.a



# Needs pthreads
find_package(Threads REQUIRED)
target_link_libraries(cpu_check Threads::Threads)

# Needs zlib #default
find_package (ZLIB REQUIRED)
# if(ZLIB_INCLUDE_DIRS)
# 	include_directories(${ZLIB_INCLUDE_DIRS})
# endif(ZLIB_INCLUDE_DIRS)
# if(ZLIB_LIBRARIES)
# 	target_link_libraries(cpu_check ${ZLIB_LIBRARIES})
#   target_link_libraries(compressor ${ZLIB_LIBRARIES})
#   target_link_libraries(hasher ${ZLIB_LIBRARIES})
# endif(ZLIB_LIBRARIES)

#Instrumented Zlib
set(ZLIB_INCLUDE_DIRS /home/NEW_GIT/qed_sdc/zlib-ng/install/include)  #mine
# set(ZLIB_INCLUDE_DIRS /home/NEW_GIT/qed_sdc/zlib-ng/build5/CMakeFiles/zlibstatic.dir)  #mine
set(ZLIB_LIBRARIES /home/NEW_GIT/qed_sdc/zlib-ng/install/lib/libz-ng.a) 
include_directories(${ZLIB_INCLUDE_DIRS})
target_link_libraries(cpu_check ${ZLIB_LIBRARIES})
target_link_libraries(compressor ${ZLIB_LIBRARIES})
target_link_libraries(hasher ${ZLIB_LIBRARIES})




# Needs OpenSSL
#Use instrumented OpenSSL
####set(OPENSSL_INCLUDE_DIRS /home/NEW_GIT/qed_sdc/openssl/include/openssl)  #mine
####set(OPENSSL_LIBRARIES /home/NEW_GIT/qed_sdc/openssl/libssl.a)  #mine
####include_directories(${OPENSSL_INCLUDE_DIRS})
####target_link_libraries(cpu_check ${OPENSSL_LIBRARIES} )
####target_link_libraries(crypto ${OPENSSL_LIBRARIES})
####target_link_libraries(hasher ${OPENSSL_LIBRARIES})


#Use default OpenSSL
find_package (OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIRS} )
target_link_libraries(cpu_check ${OPENSSL_LIBRARIES} )
target_link_libraries(crypto ${OPENSSL_LIBRARIES})
target_link_libraries(hasher ${OPENSSL_LIBRARIES})

# include_directories(${OPENSSL_INCLUDE_DIRS} /home/NEW_GIT/qed_sdc/openssl/include/crypto/)
# target_link_libraries(cpu_check ${OPENSSL_LIBRARIES} /home/NEW_GIT/qed_sdc/openssl/libcrypto.a)
# target_link_libraries(crypto ${OPENSSL_LIBRARIES} /home/NEW_GIT/qed_sdc/openssl/libcrypto.a)
# target_link_libraries(hasher ${OPENSSL_LIBRARIES} /home/NEW_GIT/qed_sdc/openssl/libcrypto.a)

# Static linking of OpenSSL may require -ldl, link it if found.
find_library (dl dl)
if(dl)
	target_link_libraries(cpu_check dl)
endif(dl)



# link malign_buffer first as it has a lot of dependencies.
target_link_libraries(malign_buffer utils)

target_link_libraries(crc32c_test crc32c /home/NEW_GIT/qed_sdc/prettyprint.o /home/NEW_GIT/qed_sdc/mmap_file.o)
target_link_libraries(compressor malign_buffer)
target_link_libraries(crypto malign_buffer)
target_link_libraries(hasher crc32c farmhash malign_buffer utils)
target_link_libraries(pattern_generator malign_buffer)
target_link_libraries(silkscreen utils)


target_link_libraries(cpu_check avx compressor crc32c crypto fvt_controller hasher malign_buffer pattern_generator silkscreen utils ${GLOB_ABSL} ${ZLIB_LIBRARIES} "-flegacy-pass-manager -Xclang -load -Xclang /home/NEW_GIT/qed_sdc/InsertClonedInstPass.so" /home/NEW_GIT/qed_sdc/prettyprint.o /home/NEW_GIT/qed_sdc/mmap_file.o -L/home/NEW_GIT/qed_sdc/llvm-project/build/projects/libc/lib -lllvmlibc -static) #/home/NEW_GIT/qed_sdc/openssl/libcrypto.a -static)

install (TARGETS cpu_check DESTINATION bin)
