cmake_minimum_required(VERSION 3.13)

project(piper C CXX)

option(BUILD_SHARED_LIBS "Build using shared libraries" OFF)

file(READ "${CMAKE_CURRENT_LIST_DIR}/VERSION" piper_version)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(MSVC)
  # Force compiler to use UTF-8 for IPA constants
  add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
  add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
elseif(NOT APPLE)
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra")
  list(APPEND CMAKE_BUILD_RPATH "$ORIGIN")
  list(APPEND CMAKE_INSTALL_RPATH "$ORIGIN")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()

add_library(piper src/cpp/piper.cpp)
add_executable(test_piper src/cpp/test.cpp src/cpp/piper.cpp)

# NOTE: external project prefix are shortened because of path length restrictions on Windows
# NOTE: onnxruntime is pulled from piper-phonemize

# ---- fmt ---

set(fmt_FOUND FALSE)

if(NOT fmt_FOUND AND TARGET "fmt::fmt")
  list(APPEND FMT_LINK_LIBRARIES "fmt::fmt")
  set(fmt_FOUND TRUE)
endif()

if(NOT fmt_FOUND AND NOT DEFINED FMT_DIR)
  find_package(fmt)
  if(fmt_FOUND)
    list(APPEND FMT_LINK_LIBRARIES "fmt::fmt")
  endif()
endif()

if(NOT fmt_FOUND AND NOT DEFINED FMT_DIR)
  set(FMT_VERSION "10.0.0")
  set(FMT_DIR "${CMAKE_CURRENT_BINARY_DIR}/fi")

  include(ExternalProject)
  ExternalProject_Add(
    fmt_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/f"
    URL "https://github.com/fmtlib/fmt/archive/refs/tags/${FMT_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${FMT_DIR}
    CMAKE_ARGS -DFMT_TEST:BOOL=OFF  # Don't build all the tests
  )
  add_dependencies(piper fmt_external)
  add_dependencies(test_piper fmt_external)
endif()

if(NOT fmt_FOUND AND DEFINED FMT_DIR)
  list(APPEND FMT_LINK_LIBRARIES "fmt")
  list(APPEND FMT_LINK_DIRECTORIES "${FMT_DIR}/lib")
  list(APPEND FMT_INCLUDE_DIRECTORIES "${FMT_DIR}/include")
  set(fmt_FOUND TRUE)
endif()

# ---- spdlog ---

set(spdlog_FOUND FALSE)

if(NOT spdlog_FOUND AND TARGET "spdlog::spdlog")
  list(APPEND SPDLOG_LINK_LIBRARIES "spdlog::spdlog")
  set(spdlog_FOUND TRUE)
endif()

if(NOT spdlog_FOUND AND NOT DEFINED SPDLOG_DIR)
  find_package(spdlog)
  if(spdlog_FOUND)
    list(APPEND SPDLOG_LINK_LIBRARIES "spdlog::spdlog")
  endif()
endif()

if(NOT spdlog_FOUND AND NOT DEFINED SPDLOG_DIR)
  set(SPDLOG_DIR "${CMAKE_CURRENT_BINARY_DIR}/si")
  set(SPDLOG_VERSION "1.12.0")
  include(ExternalProject)
  ExternalProject_Add(
    spdlog_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/s"
    URL "https://github.com/gabime/spdlog/archive/refs/tags/v${SPDLOG_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${SPDLOG_DIR}
  )
  add_dependencies(piper spdlog_external)
  add_dependencies(test_piper spdlog_external)
endif()

if(NOT spdlog_FOUND AND DEFINED SPDLOG_DIR)
  list(APPEND SPDLOG_LINK_LIBRARIES "spdlog")
  list(APPEND SPDLOG_LINK_DIRECTORIES "${SPDLOG_DIR}/lib")
  list(APPEND SPDLOG_INCLUDE_DIRECTORIES "${SPDLOG_DIR}/include")
  set(spdlog_FOUND TRUE)
endif()

# ---- piper-phonemize ---

include(FetchContent)
find_program(GIT_CMD git REQUIRED)
FetchContent_Declare(
  piper_phonemize
  GIT_REPOSITORY "https://github.com/rhasspy/piper-phonemize.git"
  GIT_TAG "bfc2e7549957829b0227c66a305d11cc88167bda" # master
  UPDATE_DISCONNECTED TRUE
  PATCH_COMMAND "${GIT_CMD}" "apply" "${CMAKE_CURRENT_SOURCE_DIR}/piper-phonemize.patch"
)
FetchContent_MakeAvailable(piper_phonemize)

# ---- Declare executable ----

if((NOT MSVC) AND (NOT APPLE))
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra")
  list(APPEND CMAKE_BUILD_RPATH "$ORIGIN")
  list(APPEND CMAKE_INSTALL_RPATH "$ORIGIN")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
  target_link_libraries(piper PRIVATE -static-libgcc -static-libstdc++)

  set(PIPER_EXTRA_LIBRARIES "pthread")
endif()

target_link_libraries(piper PRIVATE
  ${FMT_LINK_LIBRARIES}
  ${SPDLOG_LINK_LIBRARIES}
  espeak-ng
  onnxruntime
  ${PIPER_EXTRA_LIBRARIES}
  PUBLIC piper_phonemize
)

target_link_directories(piper PRIVATE
  ${FMT_LINK_DIRECTORIES}
  ${SPDLOG_LINK_DIRECTORIES}
)

set(PIPER_INTERFACE_INCLUDE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/include")
file(COPY src/cpp/piper.hpp src/cpp/json.hpp DESTINATION "${PIPER_INTERFACE_INCLUDE_DIRECTORY}")

target_include_directories(piper PRIVATE
  ${FMT_INCLUDE_DIRECTORIES}
  ${SPDLOG_INCLUDE_DIRECTORIES}
  INTERFACE "${PIPER_INTERFACE_INCLUDE_DIRECTORY}"
)

target_compile_definitions(piper PRIVATE _PIPER_VERSION=${piper_version})

target_compile_features(test_piper PUBLIC cxx_std_17)

target_include_directories(
  test_piper PUBLIC
  ${FMT_INCLUDE_DIRECTORIES}
  ${SPDLOG_INCLUDE_DIRECTORIES}
)

target_link_directories(
  test_piper PUBLIC
  ${FMT_LINK_DIRECTORIES}
  ${SPDLOG_LINK_DIRECTORIES}
)

target_link_libraries(test_piper PUBLIC
  ${FMT_LINK_LIBRARIES}
  ${SPDLOG_LINK_LIBRARIES}
  espeak-ng
  piper_phonemize
  onnxruntime
)

# ---- Declare install targets ----

install(
  TARGETS piper
  DESTINATION ${CMAKE_INSTALL_PREFIX})
