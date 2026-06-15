include(CheckCXXCompilerFlag)
include("../cmake/common.cmake")

add_compile_definitions(GGML_SCHED_MAX_COPIES=${GGML_SCHED_MAX_COPIES})

# enable libstdc++ assertions for debug builds
if (CMAKE_SYSTEM_NAME MATCHES "Linux")
    add_compile_definitions($<$<CONFIG:Debug>:_GLIBCXX_ASSERTIONS>)
    add_compile_definitions(_GNU_SOURCE)
endif()

if (GGML_FATAL_WARNINGS)
    if (CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        list(APPEND C_FLAGS   -Werror)
        list(APPEND CXX_FLAGS -Werror)
    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
        add_compile_options(/WX)
    endif()
endif()

if (GGML_ALL_WARNINGS)
    if (NOT MSVC)
        list(APPEND WARNING_FLAGS -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function)
        list(APPEND C_FLAGS       -Wshadow -Wstrict-prototypes -Wpointer-arith -Wmissing-prototypes
                                  -Werror=implicit-int -Werror=implicit-function-declaration)
        list(APPEND CXX_FLAGS     -Wmissing-declarations -Wmissing-noreturn)

        list(APPEND C_FLAGS   ${WARNING_FLAGS})
        list(APPEND CXX_FLAGS ${WARNING_FLAGS})

        ggml_get_flags(${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION})

        add_compile_options("$<$<COMPILE_LANGUAGE:C>:${C_FLAGS};${GF_C_FLAGS}>"
                            "$<$<COMPILE_LANGUAGE:CXX>:${CXX_FLAGS};${GF_CXX_FLAGS}>")
    else()
        # todo : msvc
        set(C_FLAGS   "")
        set(CXX_FLAGS "")
    endif()
endif()

# ggml

add_library(ggml-base
            ../include/ggml.h
            ../include/ggml-alloc.h
            ../include/ggml-backend.h
            ../include/ggml-cpp.h
            ../include/ggml-opt.h
            ../include/gguf.h
            ggml.c
            ggml.cpp
            ggml-alloc.c
            ggml-backend.cpp
            ggml-backend-meta.cpp
            ggml-opt.cpp
            ggml-threading.cpp
            ggml-threading.h
            ggml-quants.c
            ggml-quants.h
            gguf.cpp)

set_target_properties(ggml-base PROPERTIES
    VERSION ${GGML_VERSION}
    SOVERSION ${GGML_VERSION_MAJOR}
)

target_include_directories(ggml-base PRIVATE .)

if (GGML_SCHED_NO_REALLOC)
    target_compile_definitions(ggml-base PUBLIC GGML_SCHED_NO_REALLOC)
endif()

if (GGML_OPENMP)
    find_package(OpenMP)
    if (OpenMP_FOUND)
        set(GGML_OPENMP_ENABLED "ON" CACHE INTERNAL "")
    else()
        set(GGML_OPENMP_ENABLED "OFF" CACHE INTERNAL "")
        message(WARNING "OpenMP not found")
    endif()
else()
    set(GGML_OPENMP_ENABLED "OFF" CACHE INTERNAL "")
endif()

if (GGML_OPENMP_ENABLED)
    target_compile_definitions(ggml-base PRIVATE GGML_USE_OPENMP)
    target_link_libraries(ggml-base PRIVATE OpenMP::OpenMP_C OpenMP::OpenMP_CXX)
endif()

add_library(ggml
            ggml-backend-dl.cpp
            ggml-backend-reg.cpp)
add_library(ggml::ggml ALIAS ggml)

set_target_properties(ggml PROPERTIES
    VERSION ${GGML_VERSION}
    SOVERSION ${GGML_VERSION_MAJOR}
)

target_link_libraries(ggml PUBLIC ggml-base)

if (CMAKE_SYSTEM_NAME MATCHES "Linux")
    target_link_libraries(ggml PRIVATE dl)
endif()

function(ggml_add_backend_library backend)
    add_library(${backend} ${ARGN})
    target_link_libraries(ggml PUBLIC ${backend})
    install(TARGETS ${backend} LIBRARY)

    target_link_libraries(${backend} PRIVATE ggml-base)
    target_include_directories(${backend} PRIVATE ..)

    if (${BUILD_SHARED_LIBS})
        target_compile_definitions(${backend} PRIVATE GGML_BACKEND_BUILD)
        target_compile_definitions(${backend} PUBLIC  GGML_BACKEND_SHARED)
    endif()

    # Set versioning properties for all backend libraries
    set_target_properties(${backend} PROPERTIES
        VERSION ${GGML_VERSION}
        SOVERSION ${GGML_VERSION_MAJOR}
    )

    if(NOT GGML_AVAILABLE_BACKENDS)
        set(GGML_AVAILABLE_BACKENDS "${backend}"
            CACHE INTERNAL "List of backends for cmake package")
    else()
        list(FIND GGML_AVAILABLE_BACKENDS "${backend}" has_backend)
        if(has_backend EQUAL -1)
            set(GGML_AVAILABLE_BACKENDS "${GGML_AVAILABLE_BACKENDS};${backend}"
                CACHE INTERNAL "List of backends for cmake package")
        endif()
    endif()
endfunction()

function(ggml_add_backend backend)
    string(TOUPPER "GGML_${backend}" backend_id)
    if (${backend_id})
        string(TOLOWER "ggml-${backend}" backend_target)
        add_subdirectory(${backend_target})
        message(STATUS "Including ${backend} backend")
        string(TOUPPER "GGML_USE_${backend}" backend_use)
        target_compile_definitions(ggml PUBLIC ${backend_use})
    endif()
endfunction()

ggml_add_backend(CPU)

if (GGML_CPU)
    ggml_add_cpu_backend_variant_impl("")
endif()

ggml_add_backend(Vulkan)

foreach (target ggml-base ggml)
    target_include_directories(${target} PUBLIC    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include> $<INSTALL_INTERFACE:include>)
    target_compile_features   (${target} PRIVATE c_std_11 cxx_std_17) # don't bump
endforeach()

target_link_libraries(ggml-base PRIVATE Threads::Threads)

if (DEFINED MATH_LIBRARY)
    target_link_libraries(ggml-base PRIVATE ${MATH_LIBRARY})
elseif (NOT WIN32 AND NOT DEFINED ENV{ONEAPI_ROOT})
    target_link_libraries(ggml-base PRIVATE m)
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Android")
    target_link_libraries(ggml-base PRIVATE dl)
endif()

if(CMAKE_SYSTEM_NAME MATCHES "visionOS")
    target_compile_definitions(ggml-base PUBLIC _DARWIN_C_SOURCE)
endif()

if (BUILD_SHARED_LIBS)
    foreach (target ggml-base ggml)
        set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)
        target_compile_definitions(${target} PRIVATE GGML_BUILD)
        target_compile_definitions(${target} PUBLIC  GGML_SHARED)
    endforeach()
endif()
