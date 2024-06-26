#---------------------------------------------------------------------------#
# Copyright (c) 2018-2020 Mikhail Komarov <nemo@nil.foundation>
#
# Distributed under the Boost Software License, Version 1.0
# See accompanying file LICENSE_1_0.txt or copy at
# http://www.boost.org/LICENSE_1_0.txt
#---------------------------------------------------------------------------#

cmake_minimum_required(VERSION 3.12.4)

cmake_policy(SET CMP0028 NEW)
cmake_policy(SET CMP0042 NEW)
cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0057 NEW)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake"
     "${CMAKE_CURRENT_LIST_DIR}/cmake/packages"
     "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/share/modules/cmake")

include(CMConfig)
include(CMDeploy)
include(CMSetupVersion)

include(CheckSSE)
include(CheckAVX)

if(NOT CMAKE_WORKSPACE_NAME OR NOT ("${CMAKE_WORKSPACE_NAME}" STREQUAL "crypto3"))
    cm_workspace(crypto3)
endif()

cm_project(stream WORKSPACE_NAME ${CMAKE_WORKSPACE_NAME} LANGUAGES C CXX)

include(TargetArchitecture)
include(TargetConfiguration)
include(PlatformConfiguration)

include(CheckSSE)
include(CheckAVX)

check_sse()
check_avx()

option(CRYPTO3_STREAM_CHACHA "Build with CHACHA stream encryption support" TRUE)
option(CRYPTO3_STREAM_SALSA20 "Build with SALSA20 stream encryption support" TRUE)
option(CRYPTO3_STREAM_RC4 "Build with RC4 stream encryption support" TRUE)

set(BUILD_WITH_TARGET_ARCHITECTURE "" CACHE STRING "Target build architecture")

if(BUILD_WITH_TARGET_ARCHITECTURE)
    set(CMAKE_TARGET_ARCHITECTURE ${BUILD_WITH_TARGET_ARCHITECTURE})
else()
    target_architecture(CMAKE_TARGET_ARCHITECTURE)
endif()

if((NOT Boost_CONTAINER_FOUND OR NOT Boost_FOUND) AND NOT CMAKE_CROSSCOMPILING)
    find_package(Boost COMPONENTS REQUIRED container)
endif()

list(APPEND ${CURRENT_PROJECT_NAME}_PUBLIC_HEADERS)

list(APPEND ${CURRENT_PROJECT_NAME}_UNGROUPED_SOURCES)

if(CRYPTO3_STREAM_CHACHA)
    list(APPEND ${CURRENT_PROJECT_NAME}_CHACHA_HEADERS
         include/nil/crypto3/stream/chacha.hpp)

    list(APPEND ${CURRENT_PROJECT_NAME}_CHACHA_SOURCES
         src/chacha_policy.cpp)
    list(APPEND ${CURRENT_PROJECT_NAME}_PUBLIC_HEADERS
         ${${CURRENT_PROJECT_NAME}_CHACHA_HEADERS})
    list(APPEND ${CURRENT_PROJECT_NAME}_SOURCES
         ${${CURRENT_PROJECT_NAME}_CHACHA_SOURCES})
endif()

if(CRYPTO3_STREAM_SALSA20)
    list(APPEND ${CURRENT_PROJECT_NAME}_SALSA20_HEADERS
         include/nil/crypto3/stream/salsa20.hpp)

    list(APPEND ${CURRENT_PROJECT_NAME}_SALSA20_SOURCES
         src/salsa20_policy.cpp)

    list(APPEND ${CURRENT_PROJECT_NAME}_PUBLIC_HEADERS
         ${${CURRENT_PROJECT_NAME}_SALSA20_HEADERS})
    list(APPEND ${CURRENT_PROJECT_NAME}_SOURCES
         ${${CURRENT_PROJECT_NAME}_SALSA20_SOURCES})
endif()

if(CRYPTO3_STREAM_RC4)
    list(APPEND ${CURRENT_PROJECT_NAME}_RC4_HEADERS
         include/nil/crypto3/stream/rc4.hpp)

    list(APPEND ${CURRENT_PROJECT_NAME}_RC4_SOURCES)

    list(APPEND ${CURRENT_PROJECT_NAME}_PUBLIC_HEADERS
         ${${CURRENT_PROJECT_NAME}_RC4_HEADERS})
    list(APPEND ${CURRENT_PROJECT_NAME}_SOURCES
         ${${CURRENT_PROJECT_NAME}_RC4_SOURCES})
endif()

list(APPEND ${CURRENT_PROJECT_NAME}_HEADERS
     ${${CURRENT_PROJECT_NAME}_PUBLIC_HEADERS})

list(APPEND ${CURRENT_PROJECT_NAME}_SOURCES
     ${${CURRENT_PROJECT_NAME}_UNGROUPED_SOURCES})

cm_setup_version(VERSION 0.1.0 PREFIX ${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME})

add_library(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE)

set_target_properties(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} PROPERTIES
                      EXPORT_NAME ${CURRENT_PROJECT_NAME})

target_include_directories(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE
                           "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
                           "$<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>"

                           ${Boost_INCLUDE_DIRS})

target_link_libraries(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE
                      ${CMAKE_WORKSPACE_NAME}::block
                      ${Boost_LIBRARIES})

if(${CMAKE_TARGET_ARCHITECTURE} STREQUAL "x86_64" OR
   ${CMAKE_TARGET_ARCHITECTURE} STREQUAL "x86")
    if(CXX_HAS_AVX_2)
        target_compile_definitions(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE
                                   ${CMAKE_UPPER_WORKSPACE_NAME}_HAS_CHACHA_AVX2)
        target_compile_options(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE "-mavx2")
    elseif(SSE2_FOUND)
        target_compile_definitions(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE
                                   ${CMAKE_UPPER_WORKSPACE_NAME}_HAS_CHACHA_SSE2)
        target_compile_options(${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME} INTERFACE "-msse2")
    endif()
endif()

cm_deploy(TARGETS ${CMAKE_WORKSPACE_NAME}_${CURRENT_PROJECT_NAME}
          INCLUDE include
          NAMESPACE ${CMAKE_WORKSPACE_NAME}::)

include(CMTest)
cm_add_test_subdirectory(test)
