cmake_minimum_required(VERSION 2.8)
include(ExternalProject)
set(CIME_ROOT "${CMAKE_CURRENT_SOURCE_DIR}")

list(APPEND CMAKE_MODULE_PATH ${CIME_CMAKE_MODULE_DIRECTORY})
include(CIME_initial_setup)

project(cime_tests Fortran C)

# We rely on pio for cmake utilities like findnetcdf.cmake, so that we don't
# need to duplicate this cmake code
if (EXISTS ${SRC_ROOT}/libraries/parallelio/cmake)
  list(APPEND CMAKE_MODULE_PATH "${SRC_ROOT}/libraries/parallelio/cmake")
else()
  list(APPEND CMAKE_MODULE_PATH "${SRC_ROOT}/externals/scorpio/cmake")
endif()

include(CIME_utils)
find_package(NetCDF COMPONENTS C Fortran)
include_directories(${NetCDF_C_INCLUDE_DIRS} ${NetCDF_Fortran_INCLUDE_DIRS})

# TODO: Some of the below should be done in the relevant directories, not in
# this top level CMakeLists.

# ------------------------------------------------------------------------
# Build mct
# ------------------------------------------------------------------------
if (EXISTS ${SRC_ROOT}/libraries/mct)
  set(MCT_ROOT "${SRC_ROOT}/libraries/mct")
else()
  set(MCT_ROOT "${SRC_ROOT}/externals/mct")
endif()

if (USE_MPI_SERIAL)
  set(ENABLE_MPI_SERIAL "--enable-mpiserial")
else()
  set(ENABLE_MPI_SERIAL "")
endif()

ExternalProject_add(mct_project
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}
  SOURCE_DIR ${MCT_ROOT}
  BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/mct
  CONFIGURE_COMMAND ${MCT_ROOT}/configure ${ENABLE_MPI_SERIAL} --enable-debugging --prefix=${CMAKE_CURRENT_BINARY_DIR} CC=${CMAKE_C_COMPILER} FC=${CMAKE_Fortran_COMPILER} CFLAGS=${CFLAGS} FCFLAGS=${FFLAGS} SRCDIR=${MCT_ROOT} DEBUG="-g"
  BUILD_COMMAND $(MAKE) SRCDIR=${MCT_ROOT}
  # Leave things in <BINARY_DIR> rather than "installing", because we have
  # no need to move things around inside of the CMake binary directory. Also,
  # mpi-serial doesn't install properly in the out-of-source build
  INSTALL_COMMAND :
  )
# This copy_makefiles step is needed because mct currently doesn't support an
# out-of-source build. I am replicating what is done for the CIME system build.
ExternalProject_add_step(mct_project copy_makefiles
  DEPENDEES configure
  DEPENDERS build
  WORKING_DIRECTORY <BINARY_DIR>
  COMMAND cp -p <SOURCE_DIR>/Makefile .
  COMMAND mkdir -p mct
  COMMAND cp -p <SOURCE_DIR>/mct/Makefile mct/
  COMMAND mkdir -p mpeu
  COMMAND cp -p <SOURCE_DIR>/mpeu/Makefile mpeu/
  )
if (USE_MPI_SERIAL)
  ExternalProject_add_step(mct_project copy_mpi_serial_files
    DEPENDEES configure
    DEPENDERS build
    WORKING_DIRECTORY <BINARY_DIR>
    COMMAND mkdir -p mpi-serial
    COMMAND cp -p <SOURCE_DIR>/mpi-serial/Makefile mpi-serial/
    COMMAND cp <SOURCE_DIR>/mpi-serial/mpif.h mpi-serial/
    COMMAND cp <SOURCE_DIR>/mpi-serial/mpi.h mpi-serial/
    )
endif()

# Tell cmake to look for libraries & mod files here, because this is where we built libraries
include_directories(${CMAKE_CURRENT_BINARY_DIR}/mct/mct)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/mct/mpeu)
link_directories(${CMAKE_CURRENT_BINARY_DIR}/mct/mct)
link_directories(${CMAKE_CURRENT_BINARY_DIR}/mct/mpeu)
if (USE_MPI_SERIAL)
  # We need to list the mpi-serial include directory before system-level
  # directories so that we're sure to use mpi-serial's mpif.h instead of
  # an mpif.h from a system path.
  include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR}/mct/mpi-serial)
  link_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR}/mct/mpi-serial)
endif()

# ------------------------------------------------------------------------
# Done MCT build
# ------------------------------------------------------------------------

# Now a bunch of includes for share code.

# csm_share (we don't build it here because it seems to be built differently
# by different tests?)

if (EXISTS ${SRC_ROOT}/share/src)
  add_subdirectory(${SRC_ROOT}/share/src share_src)
  add_subdirectory(${SRC_ROOT}/components/cpl7/mct_shr mct_src)
  add_subdirectory(${SRC_ROOT}/share/unit_test_stubs/util csm_share_stubs)
  include_directories(${SRC_ROOT}/share/include)
else()
  add_subdirectory(${SRC_ROOT}/share/util csm_share)
  add_subdirectory(${SRC_ROOT}/share/unit_test_stubs/util csm_share_stubs)
  include_directories(${SRC_ROOT}/share/include)
endif()

# esmf_wrf_timemgr not built here because it depends on csm_share.
if (EXISTS ${SRC_ROOT}/share/src/esmf_wrf_timemgr)
  add_subdirectory(${SRC_ROOT}/share/src/esmf_wrf_timemgr esmf_wrf_timemgr)
  include_directories(${SRC_ROOT}/share/src/esmf_wrf_timemgr)
else()
  add_subdirectory(${SRC_ROOT}/share/esmf_wrf_timemgr esmf_wrf_timemgr)
  include_directories(${SRC_ROOT}/share/esmf_wrf_timemgr)
endif()

# Now the actual test directories.
if (EXISTS ${SRC_ROOT}/components/cpl7/driver/unit_test)
  add_subdirectory(${SRC_ROOT}/components/cpl7/driver/unit_test unit_test)
else()
  add_subdirectory(${SRC_ROOT}/driver-mct/unit_test unit_test)
endif()
add_subdirectory(${SRC_ROOT}/share/test/unit ${CMAKE_BINARY_DIR}/unittests)
