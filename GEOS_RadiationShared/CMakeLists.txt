esma_set_this ()

set (srcs
  radconstants.F90
  gettau.F90
  rad_types.F90
  rad_utils.F90
  cloud_condensate_inhomogeneity.F90
  cloud_subcol_gen.F90
  )

# ---------------------------------------------------------------------------
# Optional GT4Py/NDSL (PySHiELD) radiation CFFI bridge glue.
#
# BUILD_GEOS_RADIATION_GT4PY_INTERFACE is declared (and -DRUN_RRTMGP_GT4PY added)
# at the top-level CMakeLists.txt. The glue is built here, in the shared
# subcomponent, because BOTH GEOSsolar_GridComp and GEOSirrad_GridComp already link
# GEOS_RadiationShared -- so compiling the bind(c) Fortran interface module and the
# C shim here, and linking the embedding .so via an INTERFACE library here, makes
# the Fortran module (.mod) and the Python trampoline symbols reachable from both
# radiation drivers with no per-consumer duplication.
#
# STRUCTURAL CHOICE = Option B (reference the installed generator; vendor only the
# .f90/.c the Fortran/C compile needs). We vendor geos-rrtmgp/geos_rrtmgp_interface.
# {f90,c} (copied from PySHiELD pyshield/geos_bridge/cffi/) because they must be built
# by the GEOS Fortran/C toolchain. We do NOT vendor the CFFI generator or the Python
# marshalling/env modules: the generator's embedding code imports
# `pyshield.geos_bridge.*` by ABSOLUTE name, so the single source of truth is the
# installed pyshield package. The .so is produced by INVOKING THE INSTALLED MODULE
# (`python -m pyshield.geos_bridge.cffi.geos_rrtmgp_interface`), which keeps the
# Phase-1-validated generator authoritative and avoids a drifting second copy.
if (BUILD_GEOS_RADIATION_GT4PY_INTERFACE)

  # The vendored bind(c) interface module + C shim are compiled INTO this library.
  list (APPEND srcs
    geos-rrtmgp/geos_rrtmgp_interface.f90
    geos-rrtmgp/geos_rrtmgp_interface.c)

  # Like gtFV3, the embedding .so build wants mpiexec on a compute node, and the
  # Python interpreter that has pyshield installed (the venv) must be found here.
  find_package(MPI REQUIRED)
  find_package(Python3 COMPONENTS Interpreter REQUIRED)

  # Names mirror gtFV3's; the generator writes libgeos_rrtmgp_interface_py.so and
  # geos_rrtmgp_interface_py.h into CMAKE_CURRENT_BINARY_DIR (its CWD). The C shim
  # (geos_rrtmgp_interface.c) #includes that generated header, so the build dir is
  # added to the include path via the INTERFACE library below.
  set(GEOS_RRTMGP_INTERFACE_LIBRARY     ${CMAKE_CURRENT_BINARY_DIR}/libgeos_rrtmgp_interface_py.so)
  set(GEOS_RRTMGP_INTERFACE_HEADER_FILE ${CMAKE_CURRENT_BINARY_DIR}/geos_rrtmgp_interface_py.h)

  # Build the embedding .so by running the INSTALLED generator module (Option B).
  # Equivalent to `mpirun -np 1 python -m pyshield.geos_bridge.cffi.geos_rrtmgp_interface`.
  add_custom_command(
    OUTPUT ${GEOS_RRTMGP_INTERFACE_LIBRARY}
    COMMAND ${MPIEXEC_EXECUTABLE} ${MPIEXEC_NUMPROC_FLAG} 1
            ${Python3_EXECUTABLE} -m pyshield.geos_bridge.cffi.geos_rrtmgp_interface
    BYPRODUCTS ${GEOS_RRTMGP_INTERFACE_HEADER_FILE}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Building PySHiELD radiation CFFI interface library with Python"
    VERBATIM
    )
  add_custom_target(generate_geos_rrtmgp_interface_library
    DEPENDS ${GEOS_RRTMGP_INTERFACE_LIBRARY})

  # Install the .so into install/lib (same INTERFACE-library workaround as gtFV3).
  add_custom_command(TARGET generate_geos_rrtmgp_interface_library
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_INSTALL_PREFIX}/lib
    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${GEOS_RRTMGP_INTERFACE_LIBRARY}" ${CMAKE_INSTALL_PREFIX}/lib
    )

  # "Fake" INTERFACE target the real library can depend on + link (mirrors gtFV3).
  add_library(geos_rrtmgp_interface_py INTERFACE)
  target_include_directories(geos_rrtmgp_interface_py INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}> # generated geos_rrtmgp_interface_py.h
    $<BUILD_INTERFACE:${esma_include}/${this}>
    $<INSTALL_INTERFACE:include/${this}>
    )
  target_link_libraries(geos_rrtmgp_interface_py INTERFACE ${GEOS_RRTMGP_INTERFACE_LIBRARY})
  add_dependencies(geos_rrtmgp_interface_py generate_geos_rrtmgp_interface_library)
  install(TARGETS geos_rrtmgp_interface_py
    EXPORT ${PROJECT_NAME}-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    )
endif ()

esma_add_subdirectories(
  RRTMGP_cmake
  )

if (BUILD_GEOS_RADIATION_GT4PY_INTERFACE)
  esma_add_library (${this}
    SRCS ${srcs}
    DEPENDENCIES GEOS_Shared MAPL RRTMGP
    DEPENDENCIES geos_rrtmgp_interface_py) # link the embedding .so (Python trampolines)
  # The C shim needs mpi.h; MPI::MPI_C carries the include dirs + link.
  target_link_libraries (${this} PRIVATE MPI::MPI_C)
else ()
  esma_add_library (${this}
    SRCS ${srcs}
    DEPENDENCIES GEOS_Shared MAPL RRTMGP)
endif ()
