cmake_minimum_required(VERSION 3.22)
project(
        mktorrent
        VERSION 1.1
        LANGUAGES C
)

set(CMAKE_C_STANDARD 11)

#set(CMAKE_C_FLAGS -O2 -Wall -Wextra -Wpedantic)

option(USE_PTHREADS "Use multiple POSIX threads for calculating hashes" ON)
option(USE_OPENSSL "Use the SHA1 implementation in the OpenSSL library" ON)
option(USE_OPENSSL_DEPRECATED "Use deprecated OpenSSL functions" ON)
option(USE_LONG_OPTIONS "Enable long options" ON)
option(USE_LARGE_FILES "Needed on certain 32bit OSes (notably 32bit Linux) to support files and torrents > 2GiB" ON)
option(NO_HASH_CHECK "Disable a redundant check to see if the amount of bytes read from files while hashing matches the sum of reported file sizes" ON)
set(PROGRESS_PERIOD 200000 CACHE STRING "Progress reporting period in microseconds")
set(MAX_OPENFD 100 CACHE STRING "Maximum number of file descriptors mktorrent will open when scanning the directory")

# pthreads
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads)
unset(THREADS_PREFER_PTHREAD_FLAG)

if (NOT Threads_FOUND AND USE_PTHREADS)
    message(FATAL_ERROR "USE_PTHREADS is ON but thread library could not be found!")
else ()
    if (NOT CMAKE_USE_PTHREADS_INIT)
        message(FATAL_ERROR "Found thread library is not pthread compatible!")
    endif ()
endif ()

# OpenSSL
find_package(OpenSSL COMPONENTS Crypto)

if (NOT OPENSSL_FOUND AND USE_OPENSSL)
    message(FATAL_ERROR "USE_OPENSSL is ON but OpenSSL library could not be found!")
else ()
    if (NOT CMAKE_USE_PTHREADS_INIT)
        message(FATAL_ERROR "Found thread library is not pthread compatible!")
    endif ()
endif ()

#main.c ftw.c init.c sha1.c hash.c output.c msg.c ll.c
add_executable(mktorrent)
target_sources(
        mktorrent
        PUBLIC
        main.c
        ftw.c
        init.c
        ll.c
        msg.c
        output.c
        $<IF:$<BOOL:${USE_PTHREADS}>,hash_pthreads.c,hash.c>
        $<IF:$<BOOL:${USE_OPENSSL}>,,sha1.c> # TODO: Fix
)
target_include_directories(
        mktorrent
        PRIVATE
)
target_link_libraries(
        mktorrent
        PRIVATE
        $<$<BOOL:${USE_PTHREADS}>:Threads::Threads>
        $<$<BOOL:${USE_OPENSSL}>:OpenSSL::Crypto>
)
target_compile_definitions(
        mktorrent
        PRIVATE
        VERSION="${CMAKE_PROJECT_VERSION}"
        MAX_OPENFD=${MAX_OPENFD}
        PROGRESS_PERIOD=${PROGRESS_PERIOD}
        $<$<BOOL:${USE_PTHREADS}>:USE_PTHREADS>
        $<$<BOOL:${USE_OPENSSL}>:USE_OPENSSL>
        $<$<BOOL:${USE_LONG_OPTIONS}>:USE_LONG_OPTIONS>
        $<$<BOOL:${USE_LARGE_FILES}>:LARGEFILE_SOURCE FILE_OFFSET_BITS=64>
        $<$<BOOL:${NO_HASH_CHECK}>:NO_HASH_CHECK>
        $<$<OR:$<BOOL:${USE_OPENSSL_DEPRECATED}>,$<NOT:$<BOOL:${USE_OPENSSL}>>>:USE_OPENSSL_DEPRECATED>
)