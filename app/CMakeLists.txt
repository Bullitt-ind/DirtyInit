cmake_minimum_required(VERSION 3.22.1)
project(dfi_exploit C)

# JNI engine for com.dirty.init.DirtyInitActivity (System.loadLibrary("dfi_exploit")).
# Lives outside app/ so it can be built standalone too.
add_library(dfi_exploit SHARED ../native/dfi_exploit.c)

target_compile_options(dfi_exploit PRIVATE -O2 -Wall -march=armv8-a+crypto)
find_library(log-lib log)
target_link_libraries(dfi_exploit ${log-lib})
