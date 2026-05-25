# Copyright (c) Huawei Technologies Co., Ltd. 2025-2025. All rights reserved.
# MemFabric_Hybrid is licensed under Mulan PSL v2.
# You can use this software according to the terms and conditions of the Mulan PSL v2.
# You may obtain a copy of Mulan PSL v2 at:
#          http://license.coscl.org.cn/MulanPSL2
# THIS SOFTWARE IS PROVIDED ON AN "AS IS" BASIS, WITHOUT WARRANTIES OF ANY KIND,
# EITHER EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO NON-INFRINGEMENT,
# MERCHANTABILITY OR FIT FOR A PARTICULAR PURPOSE.
# See the Mulan PSL v2 for more details.


include_directories(
        ${PROJECT_HYBM_SRC_BASE}/include
        ${PROJECT_HYBM_SRC_BASE}/csrc/common
        ${PROJECT_HYBM_SRC_BASE}/csrc/data_operation
        ${PROJECT_HYBM_SRC_BASE}/csrc/data_operation/host
        ${PROJECT_HYBM_SRC_BASE}/csrc/driver
        ${PROJECT_HYBM_SRC_BASE}/csrc/driver/userspace
        ${PROJECT_HYBM_SRC_BASE}/csrc/driver/version
        ${PROJECT_HYBM_SRC_BASE}/csrc/entity
        ${PROJECT_HYBM_SRC_BASE}/csrc/mm
        ${PROJECT_HYBM_SRC_BASE}/csrc/transport
        ${PROJECT_HYBM_SRC_BASE}/csrc/transport/compose
        ${PROJECT_HYBM_SRC_BASE}/csrc/transport/device
        ${PROJECT_HYBM_SRC_BASE}/csrc/transport/host
        ${PROJECT_HYBM_SRC_BASE}/csrc/transport/utils
        ${PROJECT_HYBM_SRC_BASE}/csrc/ts_engine
        ${PROJECT_HYBM_SRC_BASE}/csrc/under_api
        ${PROJECT_UTIL_SRC_BASE}/ptracer/include
)

set(HYBM_PUBLIC_INCLUDE_DIR "${PROJECT_HYBM_SRC_BASE}/include")
option(BUILD_HYBM_COPY_EXTEND "Build HYBM copy extend kernel library" ON)

## add files into src variables
file(GLOB_RECURSE HYBMM_SRC_FILES
        ${PROJECT_HYBM_SRC_BASE}/csrc/*.cpp
        ${PROJECT_HYBM_SRC_BASE}/csrc/*.h
)
list(FILTER HYBMM_SRC_FILES EXCLUDE REGEX ${PROJECT_HYBM_SRC_BASE}/csrc/copy_extend/*)
list(FILTER HYBMM_SRC_FILES EXCLUDE REGEX "${PROJECT_HYBM_SRC_BASE}/csrc/driver/npu_direct_rdma/.*")
list(APPEND HYBMM_SRC_FILES ${PROJECT_UTIL_SRC_BASE}/mf_fault_injection_point.cpp)
list(APPEND HYBMM_SRC_FILES ${PROJECT_UTIL_SRC_BASE}/mf_fault_injection_point_registry.cpp)

## build the static and shared library
add_library(hybmm_objects OBJECT ${HYBMM_SRC_FILES})
target_compile_options(hybmm_objects PRIVATE -fPIC)
target_include_directories(hybmm_objects INTERFACE ${HYBM_PUBLIC_INCLUDE_DIR})

add_library(hybmm_static STATIC $<TARGET_OBJECTS:hybmm_objects> $<TARGET_OBJECTS:ptracer_object>)
set_target_properties(hybmm_static PROPERTIES OUTPUT_NAME "mf_hybm_core")
target_link_libraries(hybmm_static PUBLIC pthread dl)
target_include_directories(hybmm_static INTERFACE ${HYBM_PUBLIC_INCLUDE_DIR})

add_library(hybmm_shared SHARED $<TARGET_OBJECTS:hybmm_objects> $<TARGET_OBJECTS:ptracer_object>)
# Ensure dlopen("libxxx.so") can find libraries in the same directory.
# Without RPATH=$ORIGIN, runtime loader won't search the current .so path,
# causing "cannot open shared object file" errors after install (e.g. in Python wheel).
set_target_properties(hybmm_shared PROPERTIES
    OUTPUT_NAME "mf_hybm_core"
    BUILD_RPATH "\$ORIGIN"
    INSTALL_RPATH "\$ORIGIN"
    INSTALL_RPATH_USE_LINK_PATH TRUE
)
target_link_libraries(hybmm_shared PUBLIC pthread dl)
target_include_directories(hybmm_shared INTERFACE ${HYBM_PUBLIC_INCLUDE_DIR})

if (BUILD_HYBM_COPY_EXTEND AND XPU_TYPE STREQUAL "NPU")
    if (DEFINED ASCEND_HOME_PATH)
        set(HYBM_COPY_EXTEND_ASCEND_HOME "${ASCEND_HOME_PATH}")
    elseif (DEFINED ENV{ASCEND_HOME_PATH})
        set(HYBM_COPY_EXTEND_ASCEND_HOME "$ENV{ASCEND_HOME_PATH}")
    else ()
        set(HYBM_COPY_EXTEND_ASCEND_HOME "")
    endif ()

    set(HYBM_COPY_EXTEND_CCE_ARCH "dav-c220" CACHE STRING "CCE aicore arch for HYBM copy extend")
    set(HYBM_COPY_EXTEND_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/libmf_hybm_copy_extend.so")
    find_program(HYBM_COPY_EXTEND_BISHENG
            NAMES bisheng
            HINTS
                "${HYBM_COPY_EXTEND_ASCEND_HOME}/compiler/ccec_compiler/bin"
                "${HYBM_COPY_EXTEND_ASCEND_HOME}/aarch64-linux/ccec_compiler/bin"
                "${HYBM_COPY_EXTEND_ASCEND_HOME}/x86_64-linux/ccec_compiler/bin")

    if (HYBM_COPY_EXTEND_BISHENG)
        add_custom_command(
                OUTPUT "${HYBM_COPY_EXTEND_OUTPUT}"
                COMMAND "${HYBM_COPY_EXTEND_BISHENG}" -x asc
                        "${PROJECT_HYBM_SRC_BASE}/csrc/copy_extend/hybm_copy_kernel.cpp"
                        -shared -g -o "${HYBM_COPY_EXTEND_OUTPUT}"
                        "--cce-aicore-arch=${HYBM_COPY_EXTEND_CCE_ARCH}"
                DEPENDS "${PROJECT_HYBM_SRC_BASE}/csrc/copy_extend/hybm_copy_kernel.cpp"
                COMMENT "Building libmf_hybm_copy_extend.so by bisheng"
                VERBATIM)
        add_custom_target(hybm_copy_extend_shared ALL DEPENDS "${HYBM_COPY_EXTEND_OUTPUT}")
    else ()
        message(WARNING
                "BUILD_HYBM_COPY_EXTEND is ON, but bisheng was not found under "
                "ASCEND_HOME_PATH='${HYBM_COPY_EXTEND_ASCEND_HOME}'. "
                "Skip building libmf_hybm_copy_extend.so.")
    endif ()
endif ()

## install header files
file(GLOB HEADER_FILES ${PROJECT_HYBM_SRC_BASE}/include/*.h)
install(
        FILES ${HEADER_FILES}
        DESTINATION ${TARGET_INSTALL_DIR}/hybm/include
        PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)

## install library
install(
        TARGETS hybmm_static hybmm_shared
        DESTINATION ${TARGET_INSTALL_DIR}/hybm/lib64
        PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

if (BUILD_HYBM_COPY_EXTEND AND XPU_TYPE STREQUAL "NPU" AND HYBM_COPY_EXTEND_BISHENG)
    install(
            FILES "${HYBM_COPY_EXTEND_OUTPUT}"
            DESTINATION ${TARGET_INSTALL_DIR}/hybm/lib64
            PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
    )
endif ()
