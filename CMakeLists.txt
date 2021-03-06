# (C) Copyright 2018 UCAR.
#
# This software is licensed under the terms of the Apache Licence Version 2.0
# which can be obtained at http://www.apache.org/licenses/LICENSE-2.0.

################################################################################
# MOM6
################################################################################

cmake_minimum_required( VERSION 3.3.2 FATAL_ERROR )

project( mom6 Fortran )

set( CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH} )

set( ECBUILD_DEFAULT_BUILD_TYPE Release )
set( ENABLE_OS_TESTS           OFF CACHE BOOL "Disable OS tests" FORCE )
set( ENABLE_LARGE_FILE_SUPPORT OFF CACHE BOOL "Disable testing of large file support" FORCE )
set( ENABLE_MPI ON CACHE BOOL "Compile with MPI" )

include( ecbuild_system NO_POLICY_SCOPE )

ecbuild_requires_macro_version( 2.7 )

################################################################################
# Project
################################################################################

ecbuild_declare_project()

ecbuild_enable_fortran( REQUIRED )

set( MOM6_LINKER_LANGUAGE Fortran )

################################################################################
# Dependencies
################################################################################

# LAPACK
ecbuild_add_option( FEATURE LAPACK
                    DESCRIPTION "Linear Algebra PACKage"
                    REQUIRED_PACKAGES "LAPACK REQUIRED" )
if (NOT HAVE_LAPACK)
  ecbuild_error("LAPACK_LIBRARIES not defined")
endif()

# MPI
ecbuild_add_option( FEATURE MPI DEFAULT ON
                    DESCRIPTION "Support for MPI distributed parallelism"
                    REQUIRED_PACKAGES "MPI COMPONENTS Fortran" )
ecbuild_find_mpi( COMPONENTS Fortran REQUIRED )
ecbuild_include_mpi()
link_libraries(${MPI_Fortran_LIBRARIES})

# OpenMP
if( HAVE_OMP )
  ecbuild_enable_omp()
else()
  ecbuild_enable_ompstubs()
endif()

# NetCDF
find_package( NetCDF REQUIRED COMPONENTS Fortran )
include_directories( ${NETCDF_INCLUDE_DIRS} )

# fms
ecbuild_use_package( PROJECT fms REQUIRED )
include_directories( ${FMS_INCLUDE_DIRS} )

# gsw
ecbuild_use_package( PROJECT gsw REQUIRED )
include_directories( ${GSW_INCLUDE_DIRS} )

# ocean_bgc
ecbuild_add_option( FEATURE ENABLE_OCEAN_BGC
                    DEFAULT OFF
                    DESCRIPTION "Compile and use ocean bgc modules in MOM6" )
################################################################################
# Definitions
################################################################################

################################################################################
# Export package info
################################################################################

set( MOM6_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/config_src/dynamic
                       ${CMAKE_CURRENT_SOURCE_DIR}/src/framework
                       ${CMAKE_Fortran_MODULE_DIRECTORY} )
set( MOM6_LIBRARIES mom6 )

get_directory_property( MOM6_DEFINITIONS COMPILE_DEFINITIONS )

foreach( _tpl ${MOM6_TPLS} )
  string( TOUPPER ${_tpl} TPL )
  list( APPEND MOM6_EXTRA_DEFINITIONS   ${${TPL}_DEFINITIONS}  ${${TPL}_TPL_DEFINITIONS}  )
  list( APPEND MOM6_EXTRA_INCLUDE_DIRS  ${${TPL}_INCLUDE_DIRS} ${${TPL}_TPL_INCLUDE_DIRS} )
  list( APPEND MOM6_EXTRA_LIBRARIES     ${${TPL}_LIBRARIES}    ${${TPL}_TPL_LIBRARIES}    )
endforeach()

################################################################################
# Sources
################################################################################

include( mom6_compiler_flags )

include_directories( ${MOM6_INCLUDE_DIRS} ${MOM6_EXTRA_INCLUDE_DIRS} )

add_subdirectory( config_src )
add_subdirectory( src )

list( APPEND mom6_src_files
    ${config_src_files}
    ${src_files}
)

if (${ENABLE_OCEAN_BGC})
    add_definitions( -D_USE_GENERIC_TRACER )
    add_subdirectory( pkg/ocean_BGC )
    list( APPEND mom6_src_files ${ocean_bgc_src_files} )
endif()

ecbuild_add_library( TARGET   mom6
                     SOURCES  ${mom6_src_files}
                     LIBS     ${LAPACK_LIBRARIES} ${NETCDF_LIBRARIES}
                              ${FMS_LIBRARIES} ${GSW_LIBRARIES}
                     INCLUDES ${FMS_INCLUDES} ${GSW_INCLUDES}
                     INSTALL_HEADERS LISTED
                     LINKER_LANGUAGE ${MOM6_LINKER_LANGUAGE}
                    )

set ( mom6_exe_files
    config_src/solo_driver/MOM_driver.F90
)

ecbuild_add_executable( TARGET   mom6.x
                        SOURCES  ${mom6_exe_files}
                        LIBS     ${MOM6_LIBRARIES}
                        INCLUDES ${MOM6_INCLUDES}
                        LINKER_LANGUAGE ${MOM6_LINKER_LANGUAGE}
                       )

if(ECBUILD_INSTALL_FORTRAN_MODULES)
  install(DIRECTORY ${CMAKE_Fortran_MODULE_DIRECTORY}/${CMAKE_CFG_INTDIR} DESTINATION ${INSTALL_INCLUDE_DIR})
endif()

################################################################################
# Finalise configuration
################################################################################

# prepares a tar.gz of the sources and/or binaries
ecbuild_install_project( NAME mom6 )

# print the summary of the configuration
ecbuild_print_summary()
