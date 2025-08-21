# CMake build script for REGENIE
# Translates the original Makefile for Linux and macOS systems.
cmake_minimum_required(VERSION 3.15)
project(regenie VERSION 4.1 LANGUAGES CXX C Fortran)

# --- User-configurable Options (equivalent to Makefile variables) ---
# These can be set from the command line, e.g., -DREGENIE_STATIC_LINKING=ON
option(REGENIE_WITH_BOOST_IOSTREAM "Enable reading gzipped files via Boost Iostream" OFF)
option(REGENIE_STATIC_LINKING "Enable static linking (Linux-only)" OFF)
set(MKL_ROOT "" CACHE PATH "Path to Intel MKL root directory (e.g., /opt/intel/oneapi/mkl/latest)")
set(OPENBLAS_ROOT "" CACHE PATH "Path to OpenBLAS root directory")
set(HTSLIB_PATH "/usr/local/lib" CACHE PATH "Path to HTSlib installation directory (e.g., /usr/local)")

# --- Versioning ---
# Read the version from the VERSION file
if(EXISTS "${CMAKE_SOURCE_DIR}/VERSION")
    file(READ "${CMAKE_SOURCE_DIR}/VERSION" REGENIE_VERSION_STRING)
    string(STRIP "${REGENIE_VERSION_STRING}" REGENIE_VERSION_STRING) # Remove trailing newlines
else()
    set(REGENIE_VERSION_STRING "unknown")
endif()

# Replicate the .gz suffix logic from the Makefile
if(REGENIE_WITH_BOOST_IOSTREAM)
    set(REGENIE_VERSION_STRING "${REGENIE_VERSION_STRING}.gz")
endif()

# --- Global Compiler Settings ---
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Base compiler flags from CXXFLAGS
add_compile_options(-O3 -Wall -pedantic -ffast-math -Wno-unused-local-typedefs -Wno-deprecated-declarations -Wno-long-long -Wno-c11-extensions -fPIC)

# Platform-specific flags
if(APPLE)
    add_compile_options(-stdlib=libc++)
elseif(UNIX AND NOT APPLE) # For Linux
    # Use CMake's official way to find and enable OpenMP
    find_package(OpenMP)
    if(OpenMP_FOUND)
        message(STATUS "OpenMP found, enabling parallelization.")
        add_compile_options(${OpenMP_CXX_FLAGS})
    endif()
endif()

# --- External Libraries (Built with Make) ---
# For dependencies that have their own Makefiles, we'll use a custom target
# to invoke `make` and then link the resulting static library.

# This function simplifies creating targets for the external libs
function(add_external_make_lib lib_name)
    set(LIB_DIR "${CMAKE_SOURCE_DIR}/external_libs/${lib_name}")
    set(LIB_FILE "${LIB_DIR}/${ARGN}")

    add_custom_target(${lib_name}_build ALL
            COMMAND ${CMAKE_MAKE_PROGRAM}
            WORKING_DIRECTORY ${LIB_DIR}
            COMMENT "Building dependency: ${lib_name}"
    )

    # Create an IMPORTED library target
    add_library(${lib_name} STATIC IMPORTED GLOBAL)
    set_target_properties(${lib_name} PROPERTIES IMPORTED_LOCATION ${LIB_FILE})

    # Ensure the library is built before it's used
    add_dependencies(${lib_name} ${lib_name}_build)
endfunction()

add_external_make_lib(mvtnorm libMvtnorm.a)
add_external_make_lib(qf qf.a)
add_external_make_lib(quadpack libquad.a)
add_external_make_lib(pgenlib pgenlib.a)

# Remeta has a special argument
add_custom_target(remeta_build ALL
        COMMAND ${CMAKE_MAKE_PROGRAM} remeta.a HTSLIB_PATH=${HTSLIB_PATH}/lib
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/external_libs/remeta
        COMMENT "Building dependency: remeta"
)
add_library(remeta STATIC IMPORTED GLOBAL)
set_target_properties(remeta PROPERTIES IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/external_libs/remeta/remeta.a")
add_dependencies(remeta remeta_build)

# --- Main Executable Target ---
file(GLOB REGENIE_SOURCES "src/*.cpp")
add_executable(regenie ${REGENIE_SOURCES})

# Pass the version number as a compile definition
target_compile_definitions(regenie PRIVATE VERSION_NUMBER="${REGENIE_VERSION_STRING}")

# --- Include Directories ---
set(BGEN_PATH "${CMAKE_SOURCE_DIR}/external_libs/bgen-v1.1.7")
target_include_directories(regenie PRIVATE
        "${CMAKE_SOURCE_DIR}/external_libs/pgenlib"
        "${CMAKE_SOURCE_DIR}/external_libs/pgenlib/simde"
        "${CMAKE_SOURCE_DIR}/external_libs/pgenlib/include"
        "${CMAKE_SOURCE_DIR}/external_libs/cxxopts/include"
        "${CMAKE_SOURCE_DIR}/external_libs/LBFGSpp/include"
        "${CMAKE_SOURCE_DIR}/external_libs/eigen-3.4.0"
        "${BGEN_PATH}"
        "${BGEN_PATH}/genfile/include"
        "${BGEN_PATH}/3rd_party/boost_1_55_0"
        "${BGEN_PATH}/3rd_party/zstd-1.1.0/lib"
        "${BGEN_PATH}/db/include"
        "${BGEN_PATH}/3rd_party/sqlite3"
        "${CMAKE_SOURCE_DIR}/external_libs/remeta"
)

# --- Link Libraries and Paths ---

# Handle BLAS/LAPACK (MKL or OpenBLAS)
if(MKL_ROOT)
    message(STATUS "MKL detected, configuring for MKL.")
    target_compile_definitions(regenie PRIVATE WITH_MKL EIGEN_USE_BLAS EIGEN_USE_LAPACKE)
    target_include_directories(regenie PRIVATE ${MKL_ROOT}/include)
    if(REGENIE_STATIC_LINKING AND UNIX AND NOT APPLE)
        target_link_libraries(regenie PRIVATE
                "-Wl,--start-group"
                "${MKL_ROOT}/lib/intel64/libmkl_intel_lp64.a"
                "${MKL_ROOT}/lib/intel64/libmkl_gnu_thread.a"
                "${MKL_ROOT}/lib/intel64/libmkl_core.a"
                "-Wl,--end-group"
        )
    else()
        target_link_directories(regenie PRIVATE ${MKL_ROOT}/lib/intel64)
        target_link_libraries(regenie PRIVATE mkl_intel_lp64 mkl_gnu_thread mkl_core)
    endif()
    if(OpenMP_FOUND) # MKL needs an OpenMP runtime
        target_link_libraries(regenie PRIVATE OpenMP::OpenMP_CXX)
    endif()
elseif(OPENBLAS_ROOT)
    message(STATUS "OpenBLAS detected, configuring for OpenBLAS.")
    target_compile_definitions(regenie PRIVATE WITH_OPENBLAS EIGEN_USE_BLAS EIGEN_USE_LAPACKE)
    target_include_directories(regenie PRIVATE ${OPENBLAS_ROOT}/include)
    target_link_directories(regenie PRIVATE ${OPENBLAS_ROOT}/lib)
    target_link_libraries(regenie PRIVATE lapack lapacke openblas)
    # Add rpath for OpenBLAS so it can be found at runtime
    target_link_options(regenie PRIVATE "-Wl,-rpath,${OPENBLAS_ROOT}/lib")
endif()

# Handle Boost Iostream
if(REGENIE_WITH_BOOST_IOSTREAM)
    # FindBoost will find system-installed or Homebrew-installed boost
    find_package(Boost REQUIRED COMPONENTS iostreams)
    if(Boost_FOUND)
        message(STATUS "Boost Iostream found, enabling support.")
        target_compile_definitions(regenie PRIVATE HAS_BOOST_IOSTREAM)
        target_link_libraries(regenie PRIVATE Boost::iostreams)
    endif()
endif()

# Handle HTSlib (Linux-only as per Makefile logic)
if(HTSLIB_PATH AND UNIX AND NOT APPLE)
    message(STATUS "HTSlib path provided, configuring for HTSlib.")
    target_compile_definitions(regenie PRIVATE WITH_HTSLIB)
    find_library(HTS_LIBRARY NAMES hts PATHS ${HTSLIB_PATH}/lib)
    if(HTS_LIBRARY)
        message(STATUS "Found HTSlib at: ${HTS_LIBRARY}")
        target_link_libraries(regenie PRIVATE ${HTS_LIBRARY} z bz2 lzma curl)
    else()
        message(WARNING "HTSlib library not found in ${HTSLIB_PATH}/lib")
    endif()
endif()

# Link the pre-built BGEN library and its dependencies
target_link_directories(regenie PRIVATE
        "${BGEN_PATH}/build"
        "${BGEN_PATH}/build/3rd_party/zstd-1.1.0"
        "${BGEN_PATH}/build/db"
        "${BGEN_PATH}/build/3rd_party/sqlite3"
)
target_link_libraries(regenie PRIVATE bgen zstd db sqlite3)

# Link all the custom-built external libraries
target_link_libraries(regenie PRIVATE
        mvtnorm
        qf
        quadpack
        pgenlib
        remeta
)

# Link standard system libraries
find_package(ZLIB REQUIRED)
target_link_libraries(regenie PRIVATE ZLIB::ZLIB dl m)

# Handle static linking flags on Linux
if(REGENIE_STATIC_LINKING AND UNIX AND NOT APPLE)
    target_link_options(regenie PRIVATE "-static-libgcc" "-static-libstdc++")
endif()

# On macOS, add the Homebrew lib directory to the runtime path (rpath)
# This solves the `dyld: Library not loaded` error for gfortran, etc.
if(APPLE)
    if(EXISTS "/opt/homebrew/lib")
        target_link_options(regenie PRIVATE "-Wl,-rpath,/opt/homebrew/lib")
    elseif(EXISTS "/usr/local/lib")
        target_link_options(regenie PRIVATE "-Wl,-rpath,/usr/local/lib")
    endif()
endif()

message(STATUS "REGENIE version: ${REGENIE_VERSION_STRING}")
message(STATUS "Configuration complete. You can now build with 'make' or 'ninja'.")