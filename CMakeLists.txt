########################################################################
# CMake build script for PacBioCCS executable.
########################################################################

if (POLICY CMP0048)
    cmake_policy(SET CMP0048 OLD)
endif()

project(PacBioCCS CXX C)
cmake_minimum_required(VERSION 2.8)

# set magic variable
set(ROOT_PROJECT_NAME ${PROJECT_NAME} CACHE STRING "root project name")

# main project paths
set(PacBioCCS_RootDir           ${PacBioCCS_SOURCE_DIR})
set(PacBioCCS_ParentDir         ${PacBioCCS_RootDir}/..)
set(PacBioCCS_IncludeDir        ${PacBioCCS_RootDir}/include)
set(PacBioCCS_DocsDir           ${PacBioCCS_RootDir}/docs)
set(PacBioCCS_SourceDir         ${PacBioCCS_RootDir}/src)
set(PacBioCCS_TestsDir          ${PacBioCCS_RootDir}/tests)
set(PacBioCCS_ThirdPartyDir     ${PacBioCCS_RootDir}/third-party)

if(NOT PacBioCCS_OutputDir)
    set(PacBioCCS_OutputDir     ${PacBioCCS_RootDir})
endif()

set(PacBioCCS_BinDir            ${PacBioCCS_OutputDir}/bin)
set(PacBioCCS_LibDir            ${PacBioCCS_OutputDir}/lib)

file(MAKE_DIRECTORY             ${PacBioCCS_BinDir})
file(MAKE_DIRECTORY             ${PacBioCCS_LibDir})

# required packages
if (NOT Boost_INCLUDE_DIRS)
    find_package(Boost 1.58.0 REQUIRED)
endif()

if(NOT HTSLIB_INCLUDE_DIRS OR
   NOT HTSLIB_LIBRARIES)
    if(HTSLIB_ROOTDIR)
        find_package(htslib
                     PATHS ${HTSLIB_ROOTDIR}
                     REQUIRED)
    elseif(PacBioBAM_RootDir)
        find_package(htslib
                     PATHS ${PacBioBAM_RootDir}/../htslib
                     REQUIRED)
    elseif(Bifx_RootDir)
        find_package(htslib
                     PATHS ${Bifx_RootDir}/lib/cpp/htslib
                     REQUIRED)
    else()
        find_package(htslib
                     PATHS ${PacBioCCS_ThirdPartyDir}/htslib
                     REQUIRED)
    endif()
endif()

find_package(Threads REQUIRED)

if (NOT ZLIB_INCLUDE_DIRS OR
    NOT ZLIB_LIBRARIES)
    find_package(ZLIB REQUIRED)
endif()

# project version
set(PacBioCCS_VERSION_MAJOR 2)
set(PacBioCCS_VERSION_MINOR 0)
set(PacBioCCS_VERSION_PATCH 0)
set(PacBioCCS_VERSION
    "${PacBioCCS_VERSION_MAJOR}.${PacBioCCS_VERSION_MINOR}.${PacBioCCS_VERSION_PATCH}"
)

# Output diagnostic information if we are defined as producing an internal build.
if(NOT DIAGNOSTICS)
    set(DIAGNOSTICS 0)
endif()
add_definitions(-DDIAGNOSTICS=${DIAGNOSTICS})

# PacBio libraries
set(Bifx_RootDir                ${PacBioCCS_RootDir}/../../.. CACHE PATH "smrtanalysis/bioinformatics path")

# pbbam
if(NOT PacBioBAM_RootDir)
    set(PacBioBAM_RootDir       ${PacBioCCS_ThirdPartyDir}/pbbam CACHE PATH "pbbam source path")
endif()
set(PacBioBAM_IncludeDir        ${PacBioBAM_RootDir}/include)

# disable extra pbbam stuff for now
set(PacBioBAM_build_docs        OFF CACHE BOOL "build pbbam docs" FORCE)
set(PacBioBAM_build_tests       OFF CACHE BOOL "build pbbam tests" FORCE)
set(PacBioBAM_build_tools       OFF CACHE BOOL "build pbbam tools" FORCE)

# pbconsensus
if(NOT PacBioConsensus_RootDir)
    set(PacBioConsensus_RootDir ${PacBioCCS_ThirdPartyDir}/ConsensusCore2 CACHE PATH "ConsensusCore2 source path")
endif()
set(PacBioConsensus_IncludeDir  ${PacBioConsensus_RootDir}/include)

# third-party libraries
set(CPPOPTPARSE_RootDir         ${PacBioCCS_ThirdPartyDir}/cpp-optparse)
set(CPPOPTPARSE_IncludeDir      ${CPPOPTPARSE_RootDir})

# seqan
if(NOT SEQAN_INCLUDE_DIRS)
    set(SEQAN_INCLUDE_DIRS      ${PacBioCCS_ThirdPartyDir}/seqan/include CACHE PATH "seqan header path")
endif()

# ccs sources
file(GLOB         PacBioCCS_CPP "${PacBioCCS_SourceDir}/*.cpp")
file(GLOB_RECURSE PacBioCCS_H   "${PacBioCCS_IncludeDir}/*.h")

# cpp-optparse sources
set(CPPOPTPARSE_CPP             ${CPPOPTPARSE_RootDir}/OptionParser.cpp)
set(CPPOPTPARSE_H               ${CPPOPTPARSE_RootDir}/OptionParser.h)

# set this compiler definition to prevent unhygenic seqan behavior
add_definitions(-DSEQAN_NO_GLOBAL_EXCEPTION_HANDLER)

# shared CXX flags for src & tests
include(CheckCXXCompilerFlag)
set(PacBioCCS_CXX_FLAGS "-std=c++11 -Wall -pedantic -msse3")

# silence gmock warning
check_cxx_compiler_flag("-Wno-unused-variable" HAS_NO_UNUSED_VARIABLE)
if(HAS_NO_UNUSED_VARIABLE)
    set(PacBioCCS_CXX_FLAGS "${PacBioCCS_CXX_FLAGS} -Wno-unused-variable")
endif(HAS_NO_UNUSED_VARIABLE)

if (NOT PBBAM_INCLUDE_DIRS OR
    NOT PBBAM_LIBRARIES)
    add_subdirectory(${PacBioBAM_RootDir} external/pbbam/build EXCLUDE_FROM_ALL)
    set(PBBAM_INCLUDE_DIRS ${PacBioBAM_IncludeDir})
    set(PBBAM_LIBRARIES pbbam)
endif()

if (NOT PBCONSENSUS_INCLUDE_DIRS OR
    NOT PBCONSENSUS_LIBRARIES)
    add_subdirectory(${PacBioConsensus_RootDir} external/ConsensusCore2/build EXCLUDE_FROM_ALL)
    set(PBCONSENSUS_INCLUDE_DIRS ${PacBioConsensus_IncludeDir})
endif()

# main exe src
add_subdirectory(src)

# testing
add_subdirectory(${PacBioCCS_TestsDir})

# find the git sha1
list(APPEND CMAKE_MODULE_PATH "${PacBioConsensus_RootDir}/cmake")
include(FindGitSha1)
find_git_sha1(CCS_GIT_SHA1)
configure_file("${PacBioCCS_IncludeDir}/pacbio/ccs/Version.h.in" "${PacBioCCS_IncludeDir}/pacbio/ccs/Version.h" @ONLY)
