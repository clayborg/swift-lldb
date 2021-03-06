set(LLVM_NO_RTTI 1)

include(AddLLVM)
include(SwiftAddCustomCommandTarget)

set(output_dir "${CMAKE_BINARY_DIR}/${CMAKE_CFG_INTDIR}/bin")
set(module_cache_dir "${CMAKE_BINARY_DIR}/${CMAKE_CFG_INTDIR}/repl_swift_module_cache")

if(NOT LLDB_BUILT_STANDALONE)
  set(swift_depends swift swift-stdlib)
endif()

if(NOT LLDB_SWIFTC)
  if(NOT LLDB_BUILT_STANDALONE)
    set(LLDB_SWIFTC $<TARGET_FILE:swift>c)
  else()
    set(LLDB_SWIFTC ${LLDB_PATH_TO_SWIFT_BUILD}/bin/swiftc)
  endif()
endif()

set(swift_compile_cmd "${LLDB_SWIFTC}"
                      -resource-dir "${LLDB_PATH_TO_SWIFT_BUILD}/lib/swift"
                      -module-cache-path "${module_cache_dir}")

if(CMAKE_CROSSCOMPILING)
  set(swift_compile_cmd ${swift_compile_cmd} -target "${LLVM_HOST_TRIPLE}" -sdk "${CMAKE_SYSROOT}")
  if(CMAKE_C_COMPILER_EXTERNAL_TOOLCHAIN)
    set(swift_compile_cmd ${swift_compile_cmd} -tools-directory "${CMAKE_C_COMPILER_EXTERNAL_TOOLCHAIN}")
  endif()
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Linux")
  set(swift_compile_cmd ${swift_compile_cmd} -Xlinker -rpath -Xlinker \$ORIGIN/../lib/swift/linux)
endif()

add_custom_command_target(
    unused_var
    COMMAND "${swift_compile_cmd}"
            -o "${output_dir}/repl_swift"
            "${CMAKE_CURRENT_SOURCE_DIR}/main.swift"
    OUTPUT "${output_dir}/repl_swift"
    VERBATIM
    ALL
    CUSTOM_TARGET_NAME repl_swift
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/main.swift"
            create_swift_module_dir
            ${swift_depends})

# Ensure we get a fresh, clean module cache for the swift repl run
# before we build repl_swift.

# Note we need to support cmake 2.8, so I could not use named
# add_custom_command entries here.  This will force delete/create
# of the module cache each run of the build.  Not a big deal, but
# if we ever require cmake 3, changing these to add_custom_command
# would do this if repl_swift was out of date.
add_custom_target(
  delete_swift_module_dir
  COMMAND ${CMAKE_COMMAND} -E remove_directory "${module_cache_dir}"
  )

add_custom_target(
  create_swift_module_dir
  COMMAND ${CMAKE_COMMAND} -E make_directory "${module_cache_dir}"
  DEPENDS delete_swift_module_dir
  )

install(
  PROGRAMS "${output_dir}/repl_swift"
  DESTINATION bin)
