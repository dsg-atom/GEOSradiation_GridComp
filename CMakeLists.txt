esma_set_this()

# ---------------------------------------------------------------------------
# Optional GT4Py/NDSL (PySHiELD) radiation CFFI bridge.
#
# Mirrors the merged gtFV3 bridge's build guard
# (FVdycoreCubed_GridComp/CMakeLists.txt: option(BUILD_GEOS_GTFV3_INTERFACE OFF)
# + add_definitions(-DRUN_GTFV3)). OFF by default so a stock build is byte-for-byte
# unchanged: when OFF, none of the vendored Fortran/C glue is compiled, the embedding
# .so is never built, and -DRUN_RRTMGP_GT4PY is never defined (so the #ifdef'd call-
# outs in GEOS_IrradGridComp.F90 / GEOS_SolarGridComp.F90 compile out entirely).
#
# This option() lives at the top level (NOT in a subcomponent) and the
# add_definitions() below it runs BEFORE esma_add_library adds the SUBCOMPONENTS
# subdirectories, so -DRUN_RRTMGP_GT4PY propagates into every subcomponent compile
# (shared, solar, irrad). The .so build + INTERFACE library live in the shared
# subcomponent (GEOS_RadiationShared), which both solar and irrad already link.
#
# NOTE (Phase 3): the GT4Py core (pyshield.*) is NOT part of this repo. Building the
# embedding .so requires the PySHiELD venv's python on PATH at CMake configure time
# (Python3_EXECUTABLE must resolve the venv that has `pyshield` installed), and that
# same venv must be importable at model runtime. See GEOS_RadiationShared/CMakeLists.txt.
option(BUILD_GEOS_RADIATION_GT4PY_INTERFACE
  "Build the GT4Py/NDSL (PySHiELD) radiation CFFI bridge" OFF)
if (BUILD_GEOS_RADIATION_GT4PY_INTERFACE)
  message(STATUS "Building GEOS radiation GT4Py/NDSL (PySHiELD) CFFI interface")
  add_definitions(-DRUN_RRTMGP_GT4PY)
endif ()

set (alldirs
  GEOS_RadiationShared
  GEOSsolar_GridComp
  GEOSirrad_GridComp
  GEOSsatsim_GridComp
  )

esma_add_library (${this}
  SRCS GEOS_RadiationGridComp.F90
  SUBCOMPONENTS ${alldirs}
  DEPENDENCIES MAPL GEOS_Shared ESMF::ESMF)

