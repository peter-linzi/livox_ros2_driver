cmake_minimum_required(VERSION 3.9)

project(livox_sdk_vendor)

find_package(ament_cmake REQUIRED)

macro(build_livox_sdk)
  set(livox_sdk_REV "v2.3.1")
  set(extra_cmake_args)
  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # Bundled fmt declares char8_t under C++11; retain other -Werror checks.
    list(APPEND extra_cmake_args
      "-DCMAKE_CXX_FLAGS:STRING=-Wno-error=c++20-compat")
  endif()

  if(LIVOX_SDK_SOURCE_DIR)
    if(NOT EXISTS "${LIVOX_SDK_SOURCE_DIR}/sdk_core/include/livox_sdk.h")
      message(FATAL_ERROR "LIVOX_SDK_SOURCE_DIR must point to Livox-SDK sources")
    endif()
    set(sdk_source_args SOURCE_DIR "${LIVOX_SDK_SOURCE_DIR}")
  else()
    set(sdk_source_args
      GIT_REPOSITORY https://github.com/Livox-SDK/Livox-SDK.git
      GIT_TAG ${livox_sdk_REV}
      GIT_SHALLOW OFF)
  endif()

  get_property(multi_config GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
  if(NOT multi_config AND DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()

  include(ExternalProject)
  externalproject_add(livox-sdk-${livox_sdk_REV}
    ${sdk_source_args}
    TIMEOUT 60
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/livox-sdk_install
      -DCMAKE_POSITION_INDEPENDENT_CODE=ON
      ${extra_cmake_args}
  )

  # The external project will install to the build folder, but we'll install that on make install.
  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/livox-sdk_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}
  )
endmacro()

option(BUILTIN_livox_sdk "Force built-in version of livox-sdk" OFF)
set(LIVOX_SDK_SOURCE_DIR "" CACHE PATH "Local Livox-SDK source directory")

list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")

find_package(livox_sdk QUIET)

if(LIVOX_SDK_SOURCE_DIR)
  set(livox_sdk_FOUND FALSE)
endif()

if(NOT livox_sdk_FOUND OR BUILTIN_livox_sdk OR LIVOX_SDK_SOURCE_DIR)
  build_livox_sdk()
endif()

install(DIRECTORY cmake DESTINATION share/${PROJECT_NAME})

ament_package(CONFIG_EXTRAS livox_sdk_vendor-extras.cmake)
