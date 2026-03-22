cmake_minimum_required(VERSION 3.15)

project(rnnoise
  VERSION 0.2
  LANGUAGES C
  DESCRIPTION "RNNoise - Real-time noise suppression library (CMake port)"
)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

option(USE_EXTERNAL_MODEL "Enable external model loading (disable built-in weights)" OFF)
option(BUILD_EXAMPLES "Build rnnoise_demo test tool" ON)
option(RNNOISE_ENABLE_X86_RTCD "Enable x86 RTCD support" OFF)
option(RNNOISE_AUTO_FETCH_MODEL "Download and extract rnnoise model archive when required" ON)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/package_version")
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/package_version" _rnnoise_package_version_line REGEX "^PACKAGE_VERSION=" LIMIT_COUNT 1)
  if(_rnnoise_package_version_line)
    string(REGEX REPLACE "^PACKAGE_VERSION=\"([^\"]+)\"$" "\\1" RNNOISE_PACKAGE_VERSION_RAW "${_rnnoise_package_version_line}")
  endif()
endif()

if(NOT RNNOISE_PACKAGE_VERSION_RAW)
  set(RNNOISE_PACKAGE_VERSION_RAW "${PROJECT_VERSION}")
endif()

set(RNNOISE_PACKAGE_VERSION_FOR_CMAKE "${PROJECT_VERSION}")
if(RNNOISE_PACKAGE_VERSION_RAW MATCHES "^([0-9]+\\.[0-9]+\\.[0-9]+)")
  set(RNNOISE_PACKAGE_VERSION_FOR_CMAKE "${CMAKE_MATCH_1}")
elseif(RNNOISE_PACKAGE_VERSION_RAW MATCHES "^([0-9]+\\.[0-9]+)")
  set(RNNOISE_PACKAGE_VERSION_FOR_CMAKE "${CMAKE_MATCH_1}.0")
elseif(RNNOISE_PACKAGE_VERSION_RAW MATCHES "^([0-9]+)")
  set(RNNOISE_PACKAGE_VERSION_FOR_CMAKE "${CMAKE_MATCH_1}.0.0")
endif()

function(rnnoise_locate_model_files out_c out_h out_h_dir)
  set(_c_file "")
  set(_h_file "")
  file(GLOB_RECURSE _c_candidates LIST_DIRECTORIES FALSE "${CMAKE_CURRENT_SOURCE_DIR}/rnnoise_data.c")
  file(GLOB_RECURSE _h_candidates LIST_DIRECTORIES FALSE "${CMAKE_CURRENT_SOURCE_DIR}/rnnoise_data.h")
  if(_c_candidates)
    list(SORT _c_candidates)
    list(GET _c_candidates 0 _c_file)
  endif()
  if(_h_candidates)
    list(SORT _h_candidates)
    list(GET _h_candidates 0 _h_file)
  endif()
  if(_h_file)
    get_filename_component(_h_dir "${_h_file}" DIRECTORY)
  else()
    set(_h_dir "")
  endif()
  set(${out_c} "${_c_file}" PARENT_SCOPE)
  set(${out_h} "${_h_file}" PARENT_SCOPE)
  set(${out_h_dir} "${_h_dir}" PARENT_SCOPE)
endfunction()

function(rnnoise_try_fetch_model)
  if(NOT RNNOISE_AUTO_FETCH_MODEL)
    return()
  endif()
  if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/model_version")
    return()
  endif()

  file(READ "${CMAKE_CURRENT_SOURCE_DIR}/model_version" _model_hash)
  string(STRIP "${_model_hash}" _model_hash)
  if(NOT _model_hash)
    return()
  endif()

  set(_model_file "rnnoise_data-${_model_hash}.tar.gz")
  set(_model_path "${CMAKE_CURRENT_SOURCE_DIR}/${_model_file}")
  set(_model_url "https://media.xiph.org/rnnoise/models/${_model_file}")

  if(EXISTS "${_model_path}")
    file(SHA256 "${_model_path}" _model_archive_hash)
    if(NOT _model_archive_hash STREQUAL _model_hash)
      file(REMOVE "${_model_path}")
    endif()
  endif()

  if(NOT EXISTS "${_model_path}")
    file(DOWNLOAD
      "${_model_url}"
      "${_model_path}"
      EXPECTED_HASH "SHA256=${_model_hash}"
      SHOW_PROGRESS
      STATUS _download_status
    )
    list(GET _download_status 0 _download_code)
    list(GET _download_status 1 _download_message)
    if(NOT _download_code EQUAL 0)
      message(FATAL_ERROR "Failed to download ${_model_url}: ${_download_message}")
    endif()
  endif()

  execute_process(
    COMMAND "${CMAKE_COMMAND}" -E tar xzf "${_model_path}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    RESULT_VARIABLE _extract_result
  )
  if(NOT _extract_result EQUAL 0)
    message(FATAL_ERROR "Failed to extract ${_model_file}")
  endif()
endfunction()

rnnoise_locate_model_files(RNNOISE_MODEL_SOURCE_C RNNOISE_MODEL_HEADER RNNOISE_MODEL_HEADER_DIR)

if((NOT USE_EXTERNAL_MODEL AND (NOT RNNOISE_MODEL_SOURCE_C OR NOT RNNOISE_MODEL_HEADER)) OR (USE_EXTERNAL_MODEL AND NOT RNNOISE_MODEL_HEADER))
  rnnoise_try_fetch_model()
  rnnoise_locate_model_files(RNNOISE_MODEL_SOURCE_C RNNOISE_MODEL_HEADER RNNOISE_MODEL_HEADER_DIR)
endif()

if(USE_EXTERNAL_MODEL)
  if(NOT RNNOISE_MODEL_HEADER)
    message(FATAL_ERROR "rnnoise_data.h is required but was not found.")
  endif()
else()
  if(NOT RNNOISE_MODEL_SOURCE_C OR NOT RNNOISE_MODEL_HEADER)
    message(FATAL_ERROR "rnnoise_data.c/rnnoise_data.h are required but were not found.")
  endif()
endif()

set(RNNOISE_SOURCES
  src/denoise.c
  src/rnn.c
  src/pitch.c
  src/kiss_fft.c
  src/celt_lpc.c
  src/nnet.c
  src/nnet_default.c
  src/parse_lpcnet_weights.c
  src/rnnoise_tables.c
)

if(NOT USE_EXTERNAL_MODEL)
  list(APPEND RNNOISE_SOURCES "${RNNOISE_MODEL_SOURCE_C}")
endif()

if(RNNOISE_ENABLE_X86_RTCD)
  list(APPEND RNNOISE_SOURCES
    src/x86/x86_dnn_map.c
    src/x86/x86cpu.c
    src/x86/nnet_sse4_1.c
    src/x86/nnet_avx2.c
  )
endif()

add_library(rnnoise ${RNNOISE_SOURCES})
add_library(rnnoise::rnnoise ALIAS rnnoise)

target_include_directories(rnnoise
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${RNNOISE_MODEL_HEADER_DIR}
)

target_compile_definitions(rnnoise PRIVATE RNNOISE_BUILD)

if(USE_EXTERNAL_MODEL)
  target_compile_definitions(rnnoise PRIVATE USE_WEIGHTS_FILE)
endif()

if(RNNOISE_ENABLE_X86_RTCD)
  target_compile_definitions(rnnoise PRIVATE RNN_ENABLE_X86_RTCD CPU_INFO_BY_ASM)
  if(NOT MSVC)
    set_source_files_properties(src/x86/nnet_sse4_1.c PROPERTIES COMPILE_OPTIONS "-msse4.1")
    set_source_files_properties(src/x86/nnet_avx2.c PROPERTIES COMPILE_OPTIONS "-mavx;-mfma;-mavx2")
  endif()
endif()

if(UNIX AND NOT APPLE)
  target_link_libraries(rnnoise PRIVATE m)
endif()

set_target_properties(rnnoise PROPERTIES
  VERSION "${RNNOISE_PACKAGE_VERSION_FOR_CMAKE}"
  SOVERSION "${PROJECT_VERSION_MAJOR}"
  OUTPUT_NAME "rnnoise"
)

if(BUILD_EXAMPLES)
  add_executable(rnnoise_demo examples/rnnoise_demo.c)
  target_link_libraries(rnnoise_demo PRIVATE rnnoise)
endif()

install(TARGETS rnnoise
  EXPORT rnnoise-targets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

if(BUILD_EXAMPLES)
  install(TARGETS rnnoise_demo RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

install(FILES include/rnnoise.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
set(PACKAGE_VERSION "${RNNOISE_PACKAGE_VERSION_RAW}")
set(lrintf_lib "")
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/rnnoise.pc.in"
  "${CMAKE_CURRENT_BINARY_DIR}/rnnoise.pc"
  @ONLY
)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/rnnoise.pc" DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

set(INSTALL_CONFIGDIR "${CMAKE_INSTALL_LIBDIR}/cmake/rnnoise")

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/rnnoise-config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/rnnoise-config.cmake"
  INSTALL_DESTINATION "${INSTALL_CONFIGDIR}"
  PATH_VARS CMAKE_INSTALL_INCLUDEDIR CMAKE_INSTALL_LIBDIR
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/rnnoise-config-version.cmake"
  VERSION "${RNNOISE_PACKAGE_VERSION_FOR_CMAKE}"
  COMPATIBILITY SameMajorVersion
)

install(EXPORT rnnoise-targets
  FILE rnnoise-targets.cmake
  NAMESPACE rnnoise::
  DESTINATION "${INSTALL_CONFIGDIR}"
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/rnnoise-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/rnnoise-config-version.cmake"
  DESTINATION "${INSTALL_CONFIGDIR}"
)
