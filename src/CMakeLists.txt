cmake_minimum_required(VERSION 3.3)

project(aliceVisionSrc LANGUAGES C CXX)

# Guard against in-source builds
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "In-source builds not allowed.")
endif()

message("CMake version: ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}")
cmake_minimum_required(VERSION 3.4)

if(NOT ALICEVISION_ROOT)
  message(FATAL_ERROR "Should build from the CMakeLists.txt in the root folder.")
endif()

# Trilean option
function(trilean_option NAME DESCRIPTION DEFAULT_VALUE)
  set(${NAME} ${DEFAULT_VALUE} CACHE STRING ${DESCRIPTION})
  set(TRILEAN_VALUES "AUTO;ON;OFF")

  set_property(CACHE ${NAME} PROPERTY STRINGS "${TRILEAN_VALUES}")
  if("${${NAME}}" IN_LIST TRILEAN_VALUES)
    message(STATUS "** ${NAME}: '${${NAME}}'")
  else()
    message(FATAL_ERROR "A trilean option only accept the values: '${TRILEAN_VALUES}'")
  endif()
endfunction()

# C++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==============================================================================
# AliceVision build options
# ==============================================================================
option(ALICEVISION_BUILD_SHARED "Build AliceVision shared libs" ON)
option(ALICEVISION_BUILD_SFM "Build AliceVision SfM part" ON)
option(ALICEVISION_BUILD_MVS "Build AliceVision MVS part" ON)
option(ALICEVISION_BUILD_EXAMPLES "Build AliceVision samples applications." OFF)
option(ALICEVISION_BUILD_COVERAGE "Enable code coverage generation (gcc only)" OFF)
trilean_option(ALICEVISION_BUILD_DOC "Build AliceVision documentation" AUTO)

trilean_option(ALICEVISION_USE_OPENMP "Enable OpenMP parallelization" ON)
trilean_option(ALICEVISION_USE_CCTAG "Enable CCTAG markers" AUTO)
trilean_option(ALICEVISION_USE_POPSIFT "Enable GPU SIFT implementation" AUTO)
trilean_option(ALICEVISION_USE_OPENGV "Enable use of OpenGV algorithms" AUTO)
trilean_option(ALICEVISION_USE_ALEMBIC "Enable Alembic I/O" AUTO)
trilean_option(ALICEVISION_USE_UNCERTAINTYTE "Enable Uncertainty computation" AUTO)
trilean_option(ALICEVISION_USE_CUDA "Enable CUDA" ON)
trilean_option(ALICEVISION_USE_OPENCV "Build opencv+aliceVision samples programs" OFF)

# Since OpenCV 3, SIFT is no longer in the default modules. See
# https://github.com/itseez/opencv_contrib for more informations.
# Enable this to be able to use OpenCV SIFT in main_ComputeFeatures_OpenCV.
option(ALICEVISION_USE_OCVSIFT "Add or not OpenCV SIFT in available features" OFF)

option(ALICEVISION_USE_MESHSDFILTER "Use MeshSDFilter library (enable MeshDenoising and MeshDecimate)" ON)

option(ALICEVISION_REQUIRE_CERES_WITH_SUITESPARSE "Require Ceres with SuiteSparse (ensure best performances)" ON)

# Default build is in Release mode
if(NOT CMAKE_BUILD_TYPE AND NOT MSVC)
  set(CMAKE_BUILD_TYPE "Release")
endif()

# Set build path
set(EXECUTABLE_OUTPUT_PATH "${ALICEVISION_ROOT}/${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")
set(LIBRARY_OUTPUT_PATH "${ALICEVISION_ROOT}/${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")

# Windows specific defines
if(WIN32)
  add_definitions(-DNOMINMAX)
  add_definitions(-D_USE_MATH_DEFINES)
  if(MSVC)
    add_definitions(/bigobj)
    add_compile_options(/MP)
  endif()
endif()

# Folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "_CMakePredefinedTargets")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# There was a bug in FindCUDA fixed in cmake 3.10 that creates empty strings on the nvcc command line.
# The solution merged in cmake 3.10 use the command COMMAND_EXPAND_LISTS only available from cmake >= 3.8
# https://gitlab.kitware.com/cmake/cmake/merge_requests/1008
# So we use another solution compatible with cmake >= 3.4 proposed here:
# https://gitlab.kitware.com/cmake/cmake/issues/16411
# if cmake version < 3.10 use our hacked version of FindCUDA:
if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} VERSION_LESS 3.10)
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/compatibilityCUDA)
endif()

# Set shared or static mode
if(ALICEVISION_BUILD_SHARED)
  set(BUILD_SHARED_LIBS ON)
  if(WIN32)
    # Export all symbols from the dynamic libraries by default (avoid dllexport markup)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
  endif()
elseif()
  set(BUILD_SHARED_LIBS OFF)
endif()

# ==============================================================================
# MACRO utility
# ==============================================================================
macro(add_target_properties _target _name)
  set(_properties)
  foreach(_prop ${ARGN})
    set(_properties "${_properties} ${_prop}")
  endforeach(_prop)
  get_target_property(_old_properties ${_target} ${_name})
  if(NOT _old_properties)
    # in case it's NOTFOUND
    set(_old_properties)
  endif(NOT _old_properties)
  set_target_properties(${_target} PROPERTIES ${_name} "${_old_properties} ${_properties}")
endmacro(add_target_properties)

# ==============================================================================
# Check that submodule have been initialized and updated
# ==============================================================================
if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/CoinUtils)
  message(FATAL_ERROR
    "\n submodule(s) are missing, please update your repository:\n"
    "  > git submodule update -i\n")
endif()

# ==============================================================================
# Additional cmake find modules
# ==============================================================================
set(CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(OptimizeForArchitecture)
OptimizeForArchitecture()
set(ALICEVISION_HAVE_SSE 0)
if(SSE2_FOUND OR TARGET_ARCHITECTURE STREQUAL "native")
  if(MSVC AND NOT ${CMAKE_CL_64})
    add_definitions(/arch:SSE2)
  endif()
  set(ALICEVISION_HAVE_SSE 1)
endif()
if(UNIX AND NOT ALICEVISION_BUILD_COVERAGE)
  set(CMAKE_C_FLAGS_RELEASE "-O3")
  set(CMAKE_CXX_FLAGS_RELEASE "-O3")
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
  include(AddCompilerFlag)

  # This flag is useful as not returning from a non-void function is an error with MSVC
  AddCompilerFlag("-Werror=return-type")
  AddCompilerFlag("-Werror=switch")
  AddCompilerFlag("-Werror=return-local-addr")
endif()

# ==============================================================================
# Check C++11 support
# ==============================================================================
include(CXX11)
check_for_cxx11_compiler(CXX11_COMPILER)

if(NOT CXX11_COMPILER)
  message(FATAL_ERROR "The compiler does not support the CXX11 standard.")
endif(NOT CXX11_COMPILER)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==============================================================================
# Enable code coverage generation (only with GCC)
# ==============================================================================
if(ALICEVISION_BUILD_COVERAGE AND CMAKE_COMPILER_IS_GNUCXX)
  message("ALICEVISION_BUILD_COVERAGE enabled")
  set(CMAKE_BUILD_TYPE "Debug")
  add_definitions(--coverage -fprofile-arcs -ftest-coverage)
  set(CMAKE_EXE_LINKER_FLAGS
    "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
endif()

# ==============================================================================
# OpenMP
# ==============================================================================
if(ALICEVISION_USE_OPENMP STREQUAL "OFF")
  set(ALICEVISION_HAVE_OPENMP 0)
else() # ON OR AUTO
  find_package(OpenMP)

  if(OPENMP_FOUND)
    set(ALICEVISION_HAVE_OPENMP 1)
    message(STATUS "OpenMP found.")
  elseif(ALICEVISION_USE_OPENMP STREQUAL "ON")
    set(ALICEVISION_HAVE_OPENMP 0)
    message(SEND_ERROR "Failed to find OpenMP.")
  else()
    set(ALICEVISION_HAVE_OPENMP 0)
  endif()
endif()

if(ALICEVISION_HAVE_OPENMP)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  if(NOT MSVC)
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
      # for those using the clang with OpenMP support
      list(APPEND ALICEVISION_LIBRARY_DEPENDENCIES iomp)
    else()
    list(APPEND ALICEVISION_LIBRARY_DEPENDENCIES gomp)
    endif()
  endif()
endif()

# ==============================================================================
# Boost
# ==============================================================================
option(BOOST_NO_CXX11 "if Boost is compiled without C++11 support (as it is often the case in OS packages) this must be enabled to avoid symbol conflicts (SCOPED_ENUM)." OFF)
find_package(Boost 1.60.0 QUIET COMPONENTS atomic container date_time filesystem graph log log_setup program_options regex serialization system thread)

if(Boost_FOUND)
  message(STATUS "Boost ${Boost_LIB_VERSION} found.")
else()
  message(SEND_ERROR "Failed to find Boost.")
  message(SEND_ERROR "${Boost_ERROR_REASON}")
endif()

# Disable BOOST autolink
add_definitions(-DBOOST_ALL_NO_LIB)

# Force BOOST to use dynamic libraries (avoid link error with boost program_options)
# https://lists.boost.org/boost-users/2009/11/54015.php
add_definitions(-DBOOST_ALL_DYN_LINK)

if(BOOST_NO_CXX11)
  # Avoid link errors on boost filesystem copy_file function
  # http://stackoverflow.com/questions/35007134/c-boost-undefined-reference-to-boostfilesystemdetailcopy-file
  add_definitions(-DBOOST_NO_CXX11_SCOPED_ENUMS)
endif()

include_directories(${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIRS})
add_definitions(${Boost_DEFINITIONS})

# ==============================================================================
# OpenEXR
# ==============================================================================
find_package(OpenEXR)
if(OPENEXR_FOUND)
  message(STATUS "OpenEXR found. (Version ${OPENEXR_VERSION})")
else()
  message(SEND_ERROR "Failed to find OpenEXR.")
endif()

# ==============================================================================
# OpenImageIO
# ==============================================================================
find_package(OpenImageIO)
if(OPENIMAGEIO_FOUND)
  message(STATUS "OpenImageIO found.")
  if(UNIX)
    # Add DL dependency on linux
    set(OPENIMAGEIO_LIBRARIES "${OPENIMAGEIO_LIBRARIES};dl")
  endif()
else()
  message(SEND_ERROR "Failed to find OpenImageIO.")
endif()

# ==============================================================================
# Mosek (linear programming interface)
# ==============================================================================
set(ALICEVISION_HAVE_MOSEK 0)

if(ALICEVISION_BUILD_SFM)
  find_package(Mosek)
  if(MOSEK_FOUND)
    set(ALICEVISION_HAVE_MOSEK 1)
    set(LP_INCLUDE_DIRS
      ${MOSEK_INCLUDE}
      ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Osi/src/OsiMsk/
    )
    include_directories(${LP_INCLUDE_DIRS})
  endif()

  #Install RULES
  install(
    DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/
    DESTINATION include/aliceVision_dependencies
    COMPONENT headers
    FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h"
  )
endif()

# ==============================================================================
# Eigen
# ==============================================================================
find_package(Eigen3 3.3 REQUIRED)
if(Eigen3_FOUND OR EIGEN3_FOUND)
  # message(FATAL_ERROR "EIGEN_INCLUDE_DIR: ${EIGEN_INCLUDE_DIR}")
  include_directories(${EIGEN_INCLUDE_DIR})
  # See https://eigen.tuxfamily.org/dox/group__TopicUnalignedArrayAssert.html
  add_definitions("-DEIGEN_DONT_ALIGN_STATICALLY=1")
  add_definitions("-DEIGEN_DONT_VECTORIZE=1")
else()
  message(FATAL_ERROR " EIGEN NOT FOUND. EIGEN_INCLUDE_DIR: ${EIGEN_INCLUDE_DIR}")
endif()

# ==============================================================================
# Ceres
# ==============================================================================
# - rely on Ceres_DIR
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  message(STATUS "Trying to find package Ceres for aliceVision: ${Ceres_DIR}")
  if(ALICEVISION_REQUIRE_CERES_WITH_SUITESPARSE)
    message(STATUS "By default, Ceres required SuiteSparse to ensure best performances. if you explicitly need to build without it, you can use the option: -DALICEVISION_REQUIRE_CERES_WITH_SUITESPARSE=OFF")
    find_package(Ceres QUIET REQUIRED COMPONENTS SuiteSparse CONFIG)
  else()
    find_package(Ceres CONFIG QUIET CONFIG)
  endif()

  if(Ceres_FOUND)
    message(STATUS "Ceres include dirs ${CERES_INCLUDE_DIRS}")
    message(STATUS "Ceres libraries ${CERES_LIBRARIES}")
    if(WIN32)
        # avoid 'ERROR' macro clashing on Windows 
        add_definitions(-DGLOG_NO_ABBREVIATED_SEVERITIES)
    endif()
    include_directories(${CERES_INCLUDE_DIRS})
  else()
    message(FATAL_ERROR "External CERES not found. Not found in Ceres_DIR: ${Ceres_DIR}")
  endif()
endif()

# ==============================================================================
# Flann
# ==============================================================================
# - internal by default (flann),
# - external if FLANN_INCLUDE_DIR_HINTS and a valid Flann setup is found
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(NOT DEFINED FLANN_INCLUDE_DIR_HINTS)
    set(FLANN_INCLUDE_DIR_HINTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/flann/src/cpp)
    set(ALICEVISION_USE_INTERNAL_FLANN ON)
  endif()
  find_package(Flann QUIET)
  if(NOT FLANN_FOUND OR ALICEVISION_USE_INTERNAL_FLANN)
    set(FLANN_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/flann/src/cpp)
    set(FLANN_LIBRARY flann_cpp_s)
  endif()
endif()

# ==============================================================================
# CoinUtils
# ==============================================================================
# - internal by default (CoinUtils),
# - external if COINUTILS_INCLUDE_DIR_HINTS and a valid CoinUtils setup is found
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(NOT DEFINED COINUTILS_INCLUDE_DIR_HINTS)
    set(COINUTILS_INCLUDE_DIR_HINTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/CoinUtils/src/)
    set(ALICEVISION_USE_INTERNAL_COINUTILS ON)
  endif()
  find_package(CoinUtils QUIET)
  if(NOT COINUTILS_FOUND OR ALICEVISION_USE_INTERNAL_COINUTILS)
    set(COINUTILS_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/CoinUtils/src/)
    set(COINUTILS_LIBRARY lib_CoinUtils)
  endif()
endif()

## ==============================================================================
## Clp
## ==============================================================================
## - internal by default (Clp),
## - external if CLP_INCLUDE_DIR_HINTS and a valid Clp setup is found
## ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(NOT DEFINED CLP_INCLUDE_DIR_HINTS)
    set(CLP_INCLUDE_DIR_HINTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Clp/src/)
    set(ALICEVISION_USE_INTERNAL_CLP ON)
  endif()
  find_package(Clp QUIET)
  if(NOT CLP_FOUND OR ALICEVISION_USE_INTERNAL_CLP)
    set(CLP_INCLUDE_DIRS
       ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Clp/src/
       ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Clp/src/OsiClp/)
    set(CLP_LIBRARIES lib_clp lib_OsiClpSolver)
  endif()
endif()

# ==============================================================================
# Osi
# ==============================================================================
# - internal by default (Osi),
# - external if OSI_INCLUDE_DIR_HINTS and a valid Osi setup is found
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(NOT DEFINED OSI_INCLUDE_DIR_HINTS)
    set(OSI_INCLUDE_DIR_HINTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Osi/src/)
    set(ALICEVISION_USE_INTERNAL_OSI ON)
  endif()
  find_package(Osi QUIET)
  if(NOT OSI_FOUND OR ALICEVISION_USE_INTERNAL_OSI)
    set(OSI_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/osi_clp/Osi/src/Osi/)
    set(OSI_LIBRARY lib_Osi)
  endif()
endif()

# ==============================================================================
# Internal CLP/OSI/COINUTILS libraries
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(ALICEVISION_USE_INTERNAL_OSI AND ALICEVISION_USE_INTERNAL_CLP AND ALICEVISION_USE_INTERNAL_COINUTILS)
    add_subdirectory(dependencies/osi_clp/)
    set_property(TARGET lib_clp PROPERTY FOLDER Dependencies/Lib_clp)
    set_property(TARGET lib_CoinUtils PROPERTY FOLDER Dependencies/Lib_CoinUtils)
    set_property(TARGET lib_Osi PROPERTY FOLDER Dependencies/Lib_Osi)
    set_property(TARGET lib_OsiClpSolver PROPERTY FOLDER Dependencies/Lib_OsiClpSolver)
  endif()
endif()

# ==============================================================================
# Lemon
# ==============================================================================
# - internal by default (Lemon),
# - external if LEMON_INCLUDE_DIR_HINTS and a valid Lemon setup is found
# ==============================================================================
if(ALICEVISION_BUILD_SFM)
  if(NOT DEFINED LEMON_INCLUDE_DIR_HINTS)
    set(LEMON_INCLUDE_DIR_HINTS ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/lemon)
    set(ALICEVISION_USE_INTERNAL_LEMON ON)
  endif()
  find_package(Lemon QUIET)
  if(NOT LEMON_FOUND OR ALICEVISION_USE_INTERNAL_LEMON)
    set(LEMON_INCLUDE_DIRS
      ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/lemon
      ${CMAKE_CURRENT_BINARY_DIR}/dependencies/lemon)
    set(LEMON_LIBRARY lemon)
  endif()
endif()

# ==============================================================================
# OpenCV
# ==============================================================================
# - optional, only external and enabled only if ALICEVISION_USE_OPENCV is ON
# ==============================================================================
set(ALICEVISION_HAVE_OPENCV 0)
set(ALICEVISION_HAVE_OCVSIFT 0)

if(ALICEVISION_BUILD_SFM)
  if(NOT ALICEVISION_USE_OPENCV STREQUAL "OFF")
    find_package(OpenCV COMPONENTS core imgproc video imgcodecs videoio features2d xfeatures2d)

    if(OpenCV_FOUND)
      set(ALICEVISION_HAVE_OPENCV 1)
      message(STATUS "OpenCV found.")
    elseif(ALICEVISION_USE_OPENCV STREQUAL "ON")
      message(SEND_ERROR "Failed to find OpenCV.")
    endif()
  endif()

  if(ALICEVISION_HAVE_OPENCV)
    include_directories(${OpenCV_INCLUDE_DIRS})
    # add a definition that allows the conditional compiling
    if(ALICEVISION_USE_OCVSIFT)
      set(ALICEVISION_HAVE_OCVSIFT 1)
    endif()
  endif()
endif()

# ==============================================================================
# Alembic
# ==============================================================================
# - optional, it allows to use the classes to export data in alembic format
# ==============================================================================
set(ALICEVISION_HAVE_ALEMBIC 0)

if(ALICEVISION_BUILD_SFM) # or ALICEVISION_BUILD_MVS
  if(NOT ALICEVISION_USE_ALEMBIC STREQUAL "OFF")
    find_package(Alembic 1.7.0 CONFIG)

    if(Alembic_FOUND)
      set(ALICEVISION_HAVE_ALEMBIC 1)
      message(STATUS "Alembic version ${Alembic_VERSION} found.")
      # There is a missing include dependency in Alembic cmake export.
      add_target_properties(Alembic::Alembic PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${ILMBASE_INCLUDE_DIR}"
      )
    elseif(ALICEVISION_USE_ALEMBIC STREQUAL "ON")
      message(SEND_ERROR "Failed to find Alembic.")
    endif()
  endif()
endif()

# ==============================================================================
# CCTag
# ==============================================================================
# - optional, only external and enabled only if ALICEVISION_USE_CCTAG is ON
# ==============================================================================
set(ALICEVISION_HAVE_CCTAG 0)

if(ALICEVISION_BUILD_SFM)
  if(NOT ALICEVISION_USE_CCTAG STREQUAL "OFF")
    if(ALICEVISION_HAVE_OPENCV)
      find_package(CCTag 1.0.0 CONFIG)

      if(CCTag_FOUND)
        set(ALICEVISION_HAVE_CCTAG 1)
        message(STATUS "CCTAG ${CCTag_VERSION} found.")
      elseif(ALICEVISION_USE_CCTAG STREQUAL "ON")
        message(SEND_ERROR "Failed to find CCTAG.")
      endif()
    elseif(ALICEVISION_USE_CCTAG STREQUAL "ON")
      message(SEND_ERROR "Can't use CCTAG without OPENCV.")
    endif()
  endif()
endif()

# ==============================================================================
# PopSift
# ==============================================================================
# - optional, only external and enabled only if ALICEVISION_USE_POPSIFT is ON
# ==============================================================================
set(ALICEVISION_HAVE_POPSIFT 0)

if(ALICEVISION_BUILD_SFM)
  if(NOT ALICEVISION_USE_POPSIFT STREQUAL "OFF")
    find_package(PopSift CONFIG)

    if(PopSift_FOUND)
      set(ALICEVISION_HAVE_POPSIFT 1)
      message(STATUS "PopSIFT found.")
    elseif(ALICEVISION_USE_POPSIFT STREQUAL "ON")
      message(SEND_ERROR "Failed to find PopSIFT.")
    endif()
  endif()
endif()

# ==============================================================================
# OpenGV
# ==============================================================================
# - optional, it allows to use the generic camera PnP algorithms for rig localization
# ==============================================================================
set(ALICEVISION_HAVE_OPENGV 0)

if(ALICEVISION_BUILD_SFM)
  if(NOT ALICEVISION_USE_OPENGV STREQUAL "OFF")
    find_package(OpenGV)

    if(OPENGV_FOUND)
      set(ALICEVISION_HAVE_OPENGV 1)
      message(STATUS "OpenGV found.")
    elseif(ALICEVISION_USE_OPENGV STREQUAL "ON")
      message(SEND_ERROR "Failed to find OpenGV.")
    endif()
  endif()

  if(ALICEVISION_HAVE_OPENGV)
    include_directories(${OPENGV_INCLUDE_DIR})
    link_directories(${OPENGV_LIBRARY_DIR})
  endif()
endif()

# ==============================================================================
# UncertaintyTE
# ==============================================================================
# - optional, only external and enabled only if ALICEVISION_USE_UNCERTAINTYTE is ON
# ==============================================================================
set(ALICEVISION_HAVE_UNCERTAINTYTE 0)

if(ALICEVISION_BUILD_SFM)
  if(NOT ALICEVISION_USE_UNCERTAINTYTE STREQUAL "OFF")
    find_package(UncertaintyTE)

    if(UNCERTAINTYTE_FOUND)
      set(ALICEVISION_HAVE_UNCERTAINTYTE 1)
      message(STATUS "UncertaintyTE found.")
    elseif(ALICEVISION_USE_UNCERTAINTYTE STREQUAL "ON")
      message(SEND_ERROR "Failed to find UncertaintyTE.")
    endif()
  endif()

  if(ALICEVISION_HAVE_UNCERTAINTYTE)
    include_directories(${UNCERTAINTYTE_INCLUDE_DIR})
    link_directories(${UNCERTAINTYTE_LIBRARY_DIR})
  endif()
endif()

# ==============================================================================
# ZLIB
# ==============================================================================
if(ALICEVISION_BUILD_MVS)
  find_package(ZLIB REQUIRED)
endif()

# ==============================================================================
# GEOGRAM
# ==============================================================================
if(ALICEVISION_BUILD_MVS)
  find_package(Geogram REQUIRED)
  message(STATUS "Geogram: ${GEOGRAM_LIBRARY}, ${GEOGRAM_INCLUDE_DIR}")
endif()

# ==============================================================================
# MeshSDFilter
# ==============================================================================
# - optional, only internal and enabled only if ALICEVISION_USE_MESHSDFILTER is ON
# ==============================================================================
set(ALICEVISION_HAVE_MESHSDFILTER 0)

if(ALICEVISION_BUILD_MVS)
  if(ALICEVISION_USE_MESHSDFILTER STREQUAL "ON")
    set(ALICEVISION_HAVE_MESHSDFILTER 1)
    add_subdirectory(dependencies/MeshSDFilter)
  endif()
endif()

# ==============================================================================
# CUDA
# ==============================================================================
set(ALICEVISION_HAVE_CUDA 0)

if(NOT ALICEVISION_USE_CUDA STREQUAL "OFF")

  if(BUILD_SHARED_LIBS)
    message(STATUS "BUILD_SHARED_LIBS ON")
    # Need to declare CUDA_USE_STATIC_CUDA_RUNTIME as an option to ensure that it is not overwritten in FindCUDA.
    option(CUDA_USE_STATIC_CUDA_RUNTIME "Use the static version of the CUDA runtime library if available" OFF)
    set(CUDA_USE_STATIC_CUDA_RUNTIME OFF)
    # Workaround to force deactivation of cuda static runtime for cmake < 3.10
    set(CUDA_cudart_static_LIBRARY 0)
  else() 
    message(STATUS "BUILD_SHARED_LIBS OFF")
    option(CUDA_USE_STATIC_CUDA_RUNTIME "Use the static version of the CUDA runtime library if available" ON)
    set(CUDA_USE_STATIC_CUDA_RUNTIME ON)
  endif()

  find_package(CUDA 7.0)

  if(CUDA_FOUND)
    set(ALICEVISION_HAVE_CUDA 1)
    message(STATUS "CUDA found.")
  elseif(ALICEVISION_USE_CUDA STREQUAL "ON")
    message(SEND_ERROR "Failed to find CUDA (>= 7.0).")
  endif()
endif()

if(ALICEVISION_HAVE_CUDA)
  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "4.9.3")
      if(CUDA_VERSION VERSION_LESS "8.0")
        MESSAGE(STATUS "Found gcc >=5 and CUDA <= 7.5, adding workaround C++ flags")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_FORCE_INLINES -D_MWAITXINTRIN_H_INCLUDED -D__STRICT_ANSI__")
      endif(CUDA_VERSION VERSION_LESS "8.0")
    endif(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "4.9.3")
  endif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

  set(CUDA_SEPARABLE_COMPILATION ON)
  message("Build Mode: ${CMAKE_BUILD_TYPE}")

  set(CUDA_NVCC_FLAGS_DEBUG   "${CUDA_NVCC_FLAGS_DEBUG};-G;-g")
  set(CUDA_NVCC_FLAGS_RELEASE "${CUDA_NVCC_FLAGS_RELEASE};-O3")
  if(CUDA_VERSION VERSION_LESS 9.0) # cuda < 9.0
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode;arch=compute_20,code=sm_20")
  endif()
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode;arch=compute_30,code=sm_30")
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode;arch=compute_35,code=sm_35")
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode;arch=compute_50,code=sm_50")
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode;arch=compute_52,code=sm_52")
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode=arch=compute_52,code=compute_52")

  if(NOT CUDA_VERSION VERSION_LESS 8.0) # cuda >= 8.0
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode=arch=compute_60,code=sm_60")
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode=arch=compute_61,code=sm_61")
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-gencode=arch=compute_62,code=sm_62")
  endif()

  if(NOT CUDA_VERSION VERSION_LESS 7.0) # cuda >= 7.0
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-std=c++11")
  endif()

  # default stream legacy implies that the 0 stream synchronizes all streams
  set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};--default-stream;legacy")
  # default stream per-thread implies that each host thread has one non-synchronizing 0-stream
  # set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};--default-stream;per-thread")
  # print local memory usage per kernel: -Xptxas;-v
  # CUDA >= 7.5: -Xptxas;--warn-on-local-memory-usage;-Xptxas;--warn-on-spills
  message(STATUS "CUDA Version is ${CUDA_VERSION}")
  if(NOT CUDA_VERSION VERSION_LESS 7.5) # cuda >= 7.5
    if(UNIX)
      set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-D_FORCE_INLINES")
    endif()
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-Xptxas;--warn-on-local-memory-usage")
    set(CUDA_NVCC_FLAGS         "${CUDA_NVCC_FLAGS};-Xptxas;--warn-on-spills")
  endif()

  # library required for CUDA dynamic parallelism, forgotten by CMake 3.4
  cuda_find_library_local_first(CUDA_CUDADEVRT_LIBRARY cudadevrt "\"cudadevrt\" library")

endif()

# ==============================================================================
# Documentation
# --------------------------
# Sphinx detection
# ==============================================================================
if(ALICEVISION_BUILD_DOC STREQUAL "OFF")
  set(ALICEVISION_HAVE_DOC 0)
else()
  find_package(Sphinx)

  if (EXISTS ${SPHINX_EXECUTABLE})
    set(SPHINX_HTML_DIR "${CMAKE_CURRENT_BINARY_DIR}/htmlDoc")

    configure_file(
      "${CMAKE_CURRENT_SOURCE_DIR}/../docs/sphinx/rst/conf.py"
      "${CMAKE_CURRENT_BINARY_DIR}/conf.py"
      @ONLY
    )

    add_custom_target(doc ALL
      ${SPHINX_EXECUTABLE}
      -b html
      "${CMAKE_CURRENT_SOURCE_DIR}/../docs/sphinx/rst"
      "${SPHINX_HTML_DIR}"
      COMMENT "Building HTML documentation with Sphinx")

    set_property(TARGET doc
      PROPERTY FOLDER AliceVision
    )
    set(ALICEVISION_HAVE_DOC 1)
    message(STATUS "Sphinx found.")
  elseif(ALICEVISION_BUILD_DOC STREQUAL "ON")
    set(ALICEVISION_HAVE_DOC 0)
    message(SEND_ERROR "Failed to find Sphinx.\nSphinx need to be installed to generate the documentation")
  else()
    set(ALICEVISION_HAVE_DOC 0)
  endif()
endif()

# ==============================================================================
# Include directories
# ==============================================================================
# set the directory where all the generated files (config etc) will be placed
# ==============================================================================
set(generatedDir "${CMAKE_CURRENT_BINARY_DIR}/generated")
message("generatedDir: ${generatedDir}")

# contains the "root" directory from which including all headers
set(ALICEVISION_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

set(ALICEVISION_INCLUDE_DIRS
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${generatedDir}
  ${CMAKE_CURRENT_SOURCE_DIR}/dependencies
  ${LEMON_INCLUDE_DIRS}
  ${EIGEN_INCLUDE_DIRS}
  ${CERES_INCLUDE_DIRS}
  ${UNCERTAINTYTE_INCLUDE_DIRS}
  ${MAGMA_INCLUDE_DIRS}
  ${FLANN_INCLUDE_DIRS}
  ${LP_INCLUDE_DIRS}
  ${COINUTILS_INCLUDE_DIRS}
  ${CLP_INCLUDE_DIRS}
  ${OSI_INCLUDE_DIRS}
)

include_directories(${ALICEVISION_INCLUDE_DIRS})

# ==============================================================================
# AliceVision version
# ==============================================================================
file(STRINGS "aliceVision/version.hpp" _ALICEVISION_VERSION_HPP_CONTENTS REGEX "#define ALICEVISION_VERSION_")
foreach(v MAJOR MINOR REVISION)
  if("${_ALICEVISION_VERSION_HPP_CONTENTS}" MATCHES "#define ALICEVISION_VERSION_${v} ([0-9]+)")
    set(ALICEVISION_VERSION_${v} "${CMAKE_MATCH_1}")
  else()
    message(FATAL_ERROR "Failed to retrieve the AliceVision version the source code. Missing ALICEVISION_VERSION_${v}.")
  endif()
endforeach()
set(ALICEVISION_VERSION ${ALICEVISION_VERSION_MAJOR}.${ALICEVISION_VERSION_MINOR}.${ALICEVISION_VERSION_REVISION})

# ==============================================================================
# Information print
# ==============================================================================
message("\n")
message("** AliceVision version: " ${ALICEVISION_VERSION})
message("** Build Shared libs: " ${ALICEVISION_BUILD_SHARED})
message("** Build SfM part: " ${ALICEVISION_BUILD_SFM})
message("** Build MVS part: " ${ALICEVISION_BUILD_MVS})
message("** Build AliceVision tests: " ${ALICEVISION_BUILD_TESTS})
message("** Build AliceVision documentation: " ${ALICEVISION_HAVE_DOC})
message("** Build AliceVision samples programs: " ${ALICEVISION_BUILD_EXAMPLES})
message("** Build AliceVision+OpenCV samples programs: " ${ALICEVISION_HAVE_OPENCV})
message("** Build UncertaintyTE: " ${ALICEVISION_HAVE_UNCERTAINTYTE})
message("** Build MeshSDFilter: " ${ALICEVISION_HAVE_MESHSDFILTER})
message("** Build Alembic exporter: " ${ALICEVISION_HAVE_ALEMBIC})
message("** Enable code coverage generation: " ${ALICEVISION_BUILD_COVERAGE})
message("** Enable OpenMP parallelization: " ${ALICEVISION_HAVE_OPENMP})
message("** Use CUDA: " ${ALICEVISION_HAVE_CUDA})
message("** Use OpenCV SIFT features: " ${ALICEVISION_HAVE_OCVSIFT})
message("** Use CCTAG markers: " ${ALICEVISION_HAVE_CCTAG})
message("** Use OpenGV for rig localization: " ${ALICEVISION_HAVE_OPENGV})
message("\n")

message(STATUS "EIGEN: " ${EIGEN_VERSION} "")

if(ALICEVISION_BUILD_SFM)
  message(STATUS "CERES: " ${CERES_VERSION} "")

  if(ALICEVISION_USE_INTERNAL_FLANN)
    message(STATUS "FLANN: " ${FLANN_VERSION} " (internal)")
  else()
    message(STATUS "FLANN: " ${FLANN_VERSION} " (external)")
  endif()

  if(ALICEVISION_USE_INTERNAL_CLP)
    message(STATUS "CLP: " ${CLP_VERSION} " (internal)")
  else()
    message(STATUS "CLP: " ${CLP_VERSION} " (external)")
  endif()

  if(ALICEVISION_USE_INTERNAL_COINUTILS)
    message(STATUS "COINUTILS: " ${COINUTILS_VERSION} " (internal)")
  else()
    message(STATUS "COINUTILS: " ${COINUTILS_VERSION} " (external)")
  endif()

  if(ALICEVISION_USE_INTERNAL_OSI)
    message(STATUS "OSI: " ${OSI_VERSION} " (internal)")
  else()
    message(STATUS "OSI: " ${OSI_VERSION} " (external)")
  endif()

  if(ALICEVISION_USE_INTERNAL_LEMON)
    message(STATUS "LEMON: " ${LEMON_VERSION} " (internal)")
  else()
    message(STATUS "LEMON: " ${LEMON_VERSION} " (external)")
  endif()
endif()

message("\n")

# ==============================================================================
# AliceVision CMake Helpers
# ==============================================================================
include(Helpers)

# ==============================================================================
# Internal libraries dependencies
# ==============================================================================
add_subdirectory(dependencies)

# ==============================================================================
# AliceVision modules
# ==============================================================================

# software(s) under patent or commercial licence
# Included for research purpose only
if(ALICEVISION_BUILD_SFM)
  add_subdirectory(nonFree)
endif()

# The aliceVision library itself
add_subdirectory(aliceVision)

# aliceVision tutorial examples
if(ALICEVISION_BUILD_EXAMPLES AND ALICEVISION_BUILD_SFM)
  add_subdirectory(samples)
endif()

# Complete software(s) build on aliceVision libraries
add_subdirectory(software)


# ==============================================================================
# Install rules
# ==============================================================================

install(EXPORT aliceVision-targets DESTINATION lib)
install(EXPORT aliceVision-targets
        DESTINATION share/aliceVision/cmake FILE AliceVisionTargets.cmake)

#Adapt build include paths to install path
set(ALICEVISION_INCLUDE_DIRS
  "${ALICEVISION_INCLUDE_DIRS}"
  "${CMAKE_INSTALL_PREFIX}/include/aliceVision")

list(REMOVE_ITEM ALICEVISION_INCLUDE_DIRS ${generatedDir})
string(REGEX REPLACE
  "${CMAKE_CURRENT_SOURCE_DIR}"
  "${CMAKE_INSTALL_PREFIX}/include"
  ALICEVISION_INCLUDE_DIRS
  "${ALICEVISION_INCLUDE_DIRS}"
)

string(REGEX REPLACE
  "dependencies"
  "aliceVision_dependencies"
  ALICEVISION_INCLUDE_DIRS
  "${ALICEVISION_INCLUDE_DIRS}"
)

# Create a AliceVisionConfig.cmake file. <name>Config.cmake files are searched by
# find_package() automatically. We configure that file so that we can put any
# information we want in it, e.g. version numbers, include directories, etc.
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/AliceVisionConfig.cmake.in"
               "${CMAKE_CURRENT_BINARY_DIR}/AliceVisionConfig.cmake" @ONLY)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/AliceVisionConfig.cmake"
        DESTINATION share/aliceVision/cmake)

# create the config.hpp file containing all the preprocessor definitions
set(configfile "${generatedDir}/aliceVision/config.hpp")
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.hpp.in"
               "${configfile}" @ONLY)


install(FILES "${configfile}"
        DESTINATION "include/aliceVision")

# Add uninstall target
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
  IMMEDIATE @ONLY
)

add_custom_target(uninstall "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/../cmake/cmake_uninstall.cmake")
