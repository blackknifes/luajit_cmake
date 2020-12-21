cmake_minimum_required(VERSION 3.0)

project(luajit)

if("${CMAKE_BUILD_TYPE}" MATCHES "Rel")
    set(IsDebug false)
    set(BuildType "Release")
else()
    set(IsDebug true)
    set(BuildType "Debug")
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(Platform "x64")
else()
    set(Platform "x86")
endif()

option(BUILD_SHARED off "build dll library")

set(OUTPUT_DIR "${CMAKE_CURRENT_LIST_DIR}/bin/${Platform}/${BuildType}")
set(OBJ_DIR "${CMAKE_CACHEFILE_DIR}/obj")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${OUTPUT_DIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${OUTPUT_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${OUTPUT_DIR}")

set(SRC_DIR "${CMAKE_CURRENT_LIST_DIR}/src")

set(MINILUA_EXE "${OUTPUT_DIR}/minilua.exe")
set(BUILDVM_EXE "${OUTPUT_DIR}/buildvm.exe")

set(DASMDIR "${CMAKE_CURRENT_LIST_DIR}/dynasm")
set(DASM "${DASMDIR}/dynasm.lua")
set(ALL_LIB 
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_base.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_math.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_bit.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_string.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_table.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_io.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_os.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_package.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_debug.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_jit.c"
    "${CMAKE_CURRENT_LIST_DIR}/src/lib_ffi.c"
)

file(GLOB_RECURSE BUILDVM_SOURCES "./src/host/buildvm*.c")
file(GLOB_RECURSE LUA_SOURCES "${CMAKE_CURRENT_LIST_DIR}/src/lj_*.c" "${CMAKE_CURRENT_LIST_DIR}/src/lib_*.c")

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(DASMFLAGS -D WIN -D JIT -D FFI -D P64)
else()
    set(DASMFLAGS -D WIN -D JIT -D FFI)
endif()

set_directory_properties (PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES "${OBJ_DIR}")

add_definitions("/D_CRT_SECURE_NO_DEPRECATE")
add_definitions("/D_CRT_STDIO_INLINE=__declspec(dllexport)__inline")

add_executable(minilua "./src/host/minilua.c")

file(COPY "${SRC_DIR}/vm_x86.dasc" DESTINATION "${OBJ_DIR}")

add_custom_command(TARGET minilua POST_BUILD
    COMMAND ${MINILUA_EXE} "${DASM}" -LN ${DASMFLAGS} -o "buildvm_arch.h" "vm_x86.dasc"
    COMMENT "minilua buildvm_arch.h vm_x86.dasc"
    WORKING_DIRECTORY "${OBJ_DIR}"
)

add_executable(buildvm ${BUILDVM_SOURCES})
add_dependencies(buildvm minilua)

add_custom_command(TARGET buildvm POST_BUILD
    COMMAND ${BUILDVM_EXE} -m peobj -o "${OBJ_DIR}/lj_vm.obj"
    COMMAND ${BUILDVM_EXE} -m bcdef -o "${OBJ_DIR}/lj_bcdef.h" ${ALL_LIB}
    COMMAND ${BUILDVM_EXE} -m ffdef -o "${OBJ_DIR}/lj_ffdef.h" ${ALL_LIB}
    COMMAND ${BUILDVM_EXE} -m libdef -o "${OBJ_DIR}/lj_libdef.h" ${ALL_LIB}
    COMMAND ${BUILDVM_EXE} -m recdef -o "${OBJ_DIR}/lj_recdef.h" ${ALL_LIB}
    COMMAND ${BUILDVM_EXE} -m vmdef -o "${SRC_DIR}/jit/vmdef.lua" ${ALL_LIB}
    COMMAND ${BUILDVM_EXE} -m folddef -o "${OBJ_DIR}/lj_folddef.h" lj_opt_fold.c
    WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/src"
)
target_include_directories(buildvm PRIVATE ${OBJ_DIR} "${OBJ_DIR}" "${SRC_DIR}")


file(TOUCH "${OBJ_DIR}/lj_vm.obj")

if(${BUILD_SHARED})
    add_library(lua51 SHARED ${LUA_SOURCES} "${OBJ_DIR}/lj_vm.obj")
    target_compile_definitions("/DLUA_BUILD_AS_DLL")
else()
    add_library(lua51 STATIC ${LUA_SOURCES} "${OBJ_DIR}/lj_vm.obj")

endif(${BUILD_SHARED})
add_dependencies(lua51 buildvm)
target_include_directories(lua51 PRIVATE ${OBJ_DIR} "${OBJ_DIR}")

add_executable(luajit "${CMAKE_CURRENT_LIST_DIR}/src/luajit.c")
target_link_libraries(luajit PRIVATE lua51)
add_dependencies(luajit lua51)