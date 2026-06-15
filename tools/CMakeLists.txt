# dependencies

find_package(Threads REQUIRED)

# third-party

# ...

# flags

llama_add_compile_flags()

# Only build llama-server (and mtmd which server depends on)

add_subdirectory(mtmd)

if (LLAMA_BUILD_SERVER)
    add_subdirectory(ui)
    add_subdirectory(server)
endif()
