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

    set(HYBM_COPY_EXTEND_ASCENDC_CMAKE "")
    foreach(_ascendc_dir
            "${HYBM_COPY_EXTEND_ASCEND_HOME}/tools/tikcpp/ascendc_kernel_cmake"
            "${HYBM_COPY_EXTEND_ASCEND_HOME}/compiler/tikcpp/ascendc_kernel_cmake"
            "${HYBM_COPY_EXTEND_ASCEND_HOME}/ascendc_devkit/tikcpp/samples/cmake")
        if (EXISTS "${_ascendc_dir}/ascendc.cmake")
            set(HYBM_COPY_EXTEND_ASCENDC_CMAKE "${_ascendc_dir}/ascendc.cmake")
            break()
        endif ()
    endforeach()

    if (HYBM_COPY_EXTEND_ASCENDC_CMAKE)
        set(ASCEND_CANN_PACKAGE_PATH "${HYBM_COPY_EXTEND_ASCEND_HOME}")
        set(RUN_MODE "npu" CACHE STRING "cpu/sim/npu")
        set(SOC_VERSION "Ascend910B3" CACHE STRING "system on chip type")
        include("${HYBM_COPY_EXTEND_ASCENDC_CMAKE}")

        ascendc_library(hybm_copy_extend_shared SHARED
                ${PROJECT_HYBM_SRC_BASE}/csrc/copy_extend/hybm_copy_kernel.cpp)
        set_target_properties(hybm_copy_extend_shared PROPERTIES
            OUTPUT_NAME "mf_hybm_copy_extend"
            BUILD_RPATH "\$ORIGIN"
            INSTALL_RPATH "\$ORIGIN"
            INSTALL_RPATH_USE_LINK_PATH TRUE
        )

        set(HYBM_COPY_EXTEND_INCLUDE_DIRS
                ${PROJECT_HYBM_SRC_BASE}/csrc/copy_extend
                ${HYBM_COPY_EXTEND_ASCEND_HOME}/include
                ${HYBM_COPY_EXTEND_ASCEND_HOME}/include/aclnn
                ${HYBM_COPY_EXTEND_ASCEND_HOME}/asc/include)
        ascendc_include_directories(hybm_copy_extend_shared PRIVATE ${HYBM_COPY_EXTEND_INCLUDE_DIRS})
    else ()
        message(WARNING
                "BUILD_HYBM_COPY_EXTEND is ON, but AscendC cmake was not found under "
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

if (TARGET hybm_copy_extend_shared)
    install(
            TARGETS hybm_copy_extend_shared
            DESTINATION ${TARGET_INSTALL_DIR}/hybm/lib64
            PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
    )
endif ()
