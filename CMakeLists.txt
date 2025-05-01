cmake_minimum_required(VERSION 3.15 FATAL_ERROR) # Require a reasonably modern CMake

project(GIZMO LANGUAGES C CXX Fortran)

message(STATUS "Configuring GIZMO project...")
message(STATUS "Host System: ${CMAKE_HOST_SYSTEM_NAME}")
message(STATUS "C Compiler: ${CMAKE_C_COMPILER} ${CMAKE_C_COMPILER_VERSION}")
message(STATUS "CXX Compiler: ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "Fortran Compiler: ${CMAKE_Fortran_COMPILER} ${CMAKE_Fortran_COMPILER_VERSION}")

#--------------------------------------------------------------------------
# User Configurable Options (Replaces Config.sh / CONFIGVARS checks)
#--------------------------------------------------------------------------
option(OPENMP "Enable OpenMP parallelization" OFF)
option(CHIMES "Enable CHIMES cooling/chemistry (requires Sundials)" OFF)
option(EOS_HELMHOLTZ "Enable Helmholtz EOS (requires Fortran)" OFF)
option(COOL_GRACKLE "Enable GRACKLE cooling library" OFF)
option(PMGRID "Enable PMGRID gravity solver (requires FFTW)" OFF)
option(TURB_DRIVING_SPECTRUMGRID "Enable turbulent driving spectrum grid (requires FFTW)" OFF)
option(USE_FFTW3 "Use FFTW version 3 (default is FFTW2 if PMGRID/TURB_DRIVING_SPECTRUMGRID)" OFF)
option(DOUBLEPRECISION_FFTW "Use double-precision FFTW libraries" OFF)
option(NOTYPEPREFIX_FFTW "FFTW2 installed without type prefix (e.g., fftw_mpi vs sfftw_mpi)" OFF)
option(IO_DISABLE_HDF5 "Disable HDF5 I/O support" OFF)
option(PTHREADS_NUM_THREADS "Enable POSIX Threads support (usually for external libs)" OFF)
# Add other relevant options from Makefile OPT/CONFIGVARS checks...
option(REDUCE_TREEWALK_BRANCHING "Optimization for tree walk branching" OFF)
option(USE_MPI_IN_PLACE "Use MPI_IN_PLACE communication hints" OFF)
option(NO_ISEND_IRECV_IN_DOMAIN "Disable MPI_Isend/Irecv in domain decomposition" OFF)
option(HDF5_DISABLE_VERSION_CHECK "Disable HDF5 library version check" OFF)
option(DISABLE_ALIGNED_ALLOC "Disable use of aligned memory allocation" OFF)
option(CHIMES_USE_DOUBLE_PRECISION "Use double precision for CHIMES variables" OFF)
option(OLD_HDF5 "Flag for older HDF5 API variants" OFF) # Example from MPA config
# ... add more options as identified from the various SYSTYPE blocks or Config.sh ...

# Determine if FFTW is needed at all
set(NEED_FFTW OFF)
if(PMGRID OR TURB_DRIVING_SPECTRUMGRID)
    set(NEED_FFTW ON)
endif()

# --- Configuration ---
# Specify the path to your GIZMO_config.h file
# It's often good practice to locate it relative to the CMakeLists.txt
set(GIZMO_CONFIG_HEADER "${CMAKE_CURRENT_SOURCE_DIR}/GIZMO_config.h")
message(STATUS "Looking for config header at: ${GIZMO_CONFIG_HEADER}")

# --- Read and Parse Header File ---
if(NOT EXISTS "${GIZMO_CONFIG_HEADER}")
    message(FATAL_ERROR "GIZMO_config.h not found at ${GIZMO_CONFIG_HEADER}")
else()
    # Read the file into a list of lines, skipping empty lines
    file(STRINGS "${GIZMO_CONFIG_HEADER}" GIZMO_CONFIG_LINES)

    set(GIZMO_COMPILE_DEFINITIONS "") # Initialize an empty list/string

    message(STATUS "Found defines in ${GIZMO_CONFIG_HEADER}:")
    # Loop through each line read from the file
    foreach(CURRENT_LINE ${GIZMO_CONFIG_LINES})
        # Trim leading/trailing whitespace from the line
        string(STRIP "${CURRENT_LINE}" TRIMMED_LINE)

        if(TRIMMED_LINE MATCHES "^#define[ \t]+([A-Za-z_][A-Za-z0-9_]*)([ \t]+([0-9]+))?$")
            set(DEFINE_NAME "${CMAKE_MATCH_1}")
            set(DEFINE_VALUE "${CMAKE_MATCH_3}") # Optional, if you want to capture the value

            if (DEFINE_VALUE)
                message(STATUS "  - ${DEFINE_NAME} (Value: ${DEFINE_VALUE})")
                set(${DEFINE_NAME} ${DEFINE_VALUE} CACHE INTERNAL "Define parsed from GIZMO_config.h")
                list(APPEND GIZMO_COMPILE_DEFINITIONS "${DEFINE_NAME}=${DEFINE_VALUE}")
            else()
                set(${DEFINE_NAME} TRUE CACHE INTERNAL "Define parsed from GIZMO_config.h")
                message(STATUS "  - ${DEFINE_NAME} (CMake variable set to TRUE)")
                list(APPEND GIZMO_COMPILE_DEFINITIONS "${DEFINE_NAME}")
            endif()
        endif()
    endforeach()

    # Add all collected defines as compile definitions globally
    # Alternatively, use target_compile_definitions for specific targets
    # if(GIZMO_COMPILE_DEFINITIONS)
    #     add_compile_definitions(${GIZMO_COMPILE_DEFINITIONS})
    #     message(STATUS "Added compile definitions: ${GIZMO_COMPILE_DEFINITIONS}")
    # else()
    #     message(STATUS "No defines matching the pattern found in ${GIZMO_CONFIG_HEADER}.")
    # endif()

endif()

#--------------------------------------------------------------------------
# Build Information (Git)
#--------------------------------------------------------------------------
find_package(Git QUIET)
if(GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
    execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    OUTPUT_VARIABLE GIT_COMMIT_HASH OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
    execute_process(COMMAND ${GIT_EXECUTABLE} config --get remote.origin.url
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    OUTPUT_VARIABLE GIT_REPO_URL OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
    execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    OUTPUT_VARIABLE GIT_BRANCH OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
endif()
# Provide defaults if Git info not found
if(NOT GIT_COMMIT_HASH)
  set(GIT_COMMIT_HASH "Unknown")
endif()
if(NOT GIT_REPO_URL)
  set(GIT_REPO_URL "Unknown")
endif()
if(NOT GIT_BRANCH)
  set(GIT_BRANCH "Unknown")
endif()

# Get Hostname and Username (Platform dependent)
if(CMAKE_HOST_UNIX)
  execute_process(COMMAND hostname OUTPUT_VARIABLE HOSTNAME OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
  execute_process(COMMAND whoami OUTPUT_VARIABLE USERNAME OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
else()
  set(HOSTNAME "UnknownHost")
  set(USERNAME "UnknownUser")
endif()

if(NOT HOSTNAME)
    set(HOSTNAME "UnknownHost")
endif()
if(NOT USERNAME)
    set(USERNAME "UnknownUser")
endif()

string(TIMESTAMP BUILD_TIMESTAMP "%Y-%m-%d %H:%M:%S UTC" UTC)
set(BUILDINFO_STRING "Build on ${HOSTNAME} by ${USERNAME} from ${GIT_BRANCH}:${GIT_COMMIT_HASH} at ${GIT_REPO_URL} on ${BUILD_TIMESTAMP}")
message(STATUS "Build Info String: ${BUILDINFO_STRING}")

#--------------------------------------------------------------------------
# Find Dependencies
#--------------------------------------------------------------------------
# --- MPI ---
# MPI is fundamental
find_package(MPI REQUIRED)
message(STATUS "MPI C Compiler: ${MPI_C_COMPILER}")
message(STATUS "MPI CXX Compiler: ${MPI_CXX_COMPILER}")
message(STATUS "MPI Fortran Compiler: ${MPI_Fortran_COMPILER}") # May be empty if Fortran not used/found well

# --- GSL ---
# GSL seems generally required
find_package(GSL REQUIRED)
message(STATUS "GSL Found: ${GSL_FOUND}")
message(STATUS "GSL Include Dirs: ${GSL_INCLUDE_DIRS}")
message(STATUS "GSL Libraries: ${GSL_LIBRARIES}")

# --- HDF5 (Conditional) ---
if(NOT IO_DISABLE_HDF5)
    find_package(HDF5 REQUIRED COMPONENTS C HL) # Need C and HighLevel API
    message(STATUS "HDF5 Found: ${HDF5_FOUND}")
    message(STATUS "HDF5 Include Dirs: ${HDF5_INCLUDE_DIRS}")
    message(STATUS "HDF5 Libraries: ${HDF5_LIBRARIES}")
    set(HDF5_IS_ENABLED TRUE)
else()
    message(STATUS "HDF5 support explicitly disabled.")
    set(HDF5_IS_ENABLED FALSE)
endif()

# --- FFTW (Conditional and Complex) ---
if(NEED_FFTW)
    message(STATUS "FFTW library is required.")
    set(FFTW_PRECISION_SUFFIX "f") # Default to single precision ('f') suffix for manual link
    set(FFTW_COMPILE_DEF "")
    if(DOUBLEPRECISION_FFTW)
        # For FFTW3, double precision often doesn't use a suffix for standard installs.
        # FindFFTW3 handles this. For manual linking, we assume no suffix for double.
        set(FFTW_PRECISION_SUFFIX "")
        set(FFTW_COMPILE_DEF "DOUBLEPRECISION_FFTW")
        message(STATUS "FFTW: Requesting double precision.")
    else()
        message(STATUS "FFTW: Requesting single precision.")
    endif()

    if(USE_FFTW3)
        message(STATUS "FFTW: Using FFTW3.")
        # Attempt standard find_package first, requesting MPI component
        
        if (NOT DEFINED FFTW3_ROOT)
            # If FFTW3_ROOT is not set, use CMAKE_PREFIX_PATH to help find_package
            find_package(FFTW3 QUIET COMPONENTS MPI) # Run quietly initially

            if (FFTW3_FOUND)
                # Standard find_package succeeded
                message(STATUS "FFTW3 found via find_package.")
                message(STATUS "FFTW3 Include Dirs: ${FFTW3_INCLUDE_DIRS}")
                message(STATUS "FFTW3 Libraries: ${FFTW3_LIBRARIES}")
                set(FFTW_INCLUDE_DIRS ${FFTW3_INCLUDE_DIRS})
                set(FFTW_LIBRARIES ${FFTW3_LIBRARIES})
            else()
                message(FATAL_ERROR "find_package(FFTW3 COMPONENTS MPI) failed. Ensure FFTW3 is installed or FFTW3_ROOT is set to manually configure.")
            endif()
        else()
            # find_package failed, attempt manual linking as requested
            message(WARNING "Attempting manual link with MPI, Core, and Threads FFTW3 libraries. Include path must be findable.")

            # Determine library names based on precision for manual link attempt
            # Note: GIZMO likely needs the MPI variants primarily.
            set(FFTW3_MPI_LIB_NAME "fftw3${FFTW_PRECISION_SUFFIX}_mpi")
            set(FFTW3_CORE_LIB_NAME "fftw3${FFTW_PRECISION_SUFFIX}")
            set(FFTW3_THREADS_LIB_NAME "fftw3${FFTW_PRECISION_SUFFIX}_threads") # As requested

            message(STATUS "FFTW3: Manual link library names (tentative): ${FFTW3_MPI_LIB_NAME}, ${FFTW3_CORE_LIB_NAME}, ${FFTW3_THREADS_LIB_NAME}")

            # We *must* find the include directory, even for manual linking
            find_path(FFTW3_INCLUDE_DIR_MANUAL NAMES fftw3.h
                      HINTS ENV FFTW3_ROOT ${FFTW3_ROOT} PATH_SUFFIXES include) # Help find_path
            if(NOT FFTW3_INCLUDE_DIR_MANUAL)
                message(FATAL_ERROR "FFTW3 include directory (containing fftw3.h) not found even for manual link. Cannot proceed. Set FFTW3_ROOT or CMAKE_PREFIX_PATH.")
            endif()
            message(STATUS "FFTW3 Include Dir (manual): ${FFTW3_INCLUDE_DIR_MANUAL}")
            set(FFTW_INCLUDE_DIRS ${FFTW3_INCLUDE_DIR_MANUAL})

            # Set libraries for manual linking - Order: MPI, Core, Threads (adjust if linking issues arise)
            # Pass names directly to the linker (e.g., -lfftw3f_mpi -lfftw3f -lfftw3f_threads)
            set(FFTW_LIBRARIES "${FFTW3_MPI_LIB_NAME};${FFTW3_CORE_LIB_NAME};${FFTW3_THREADS_LIB_NAME}")

            # Mark as "found" conceptually, allowing subsequent logic to use the variables
            set(FFTW3_FOUND TRUE) # Use this carefully, it bypasses find_package checks
            message(WARNING "FFTW3 linking manually set to link against: ${FFTW_LIBRARIES}. Ensure these libraries exist and are compatible (MPI, precision). Build may fail otherwise.")
        endif() # End of FFTW3_FOUND check

        # Common steps if FFTW3 is used (either found or manually set)

    else() # USE_FFTW3 is OFF, so use FFTW2 logic
        message(STATUS "FFTW: Using FFTW2.")
        # FFTW2 finding is non-standard. Rely on CMAKE_PREFIX_PATH or FFTW2_ROOT hint.
        find_path(FFTW2_INCLUDE_DIR NAMES fftw_mpi.h
                  HINTS ENV FFTW2_ROOT ${FFTW2_ROOT} PATH_SUFFIXES include) # Help find_path
        if(NOT FFTW2_INCLUDE_DIR)
            message(FATAL_ERROR "FFTW2 include dir (containing fftw_mpi.h) not found. Set FFTW2_ROOT or add to CMAKE_PREFIX_PATH.")
        endif()

        # Determine FFTW2 library name prefixes based on options
        set(FFTW2_LIB_PREFIX_R "r")
        set(FFTW2_LIB_PREFIX "")
        if(NOT NOTYPEPREFIX_FFTW)
             if(DOUBLEPRECISION_FFTW)
                 set(FFTW2_LIB_PREFIX_R "ldr")
                 set(FFTW2_LIB_PREFIX "ld")
             else() # Single precision with prefix
                 set(FFTW2_LIB_PREFIX_R "lsr")
                 set(FFTW2_LIB_PREFIX "ls")
             endif()
        else() # No type prefix
             list(APPEND COMPILE_DEFINITIONS_TO_ADD NOTYPEPREFIX_FFTW)
             message(STATUS "FFTW2: Assuming no type prefix.")
             if(DOUBLEPRECISION_FFTW)
                 # Double precision names might vary without prefix too (e.g., d vs ld)
                 set(FFTW2_LIB_PREFIX_R "dr")
                 set(FFTW2_LIB_PREFIX "d")
             endif()
             # Single precision without prefix uses default "r" and ""
        endif()

        # Find the required FFTW2 library variants using the determined prefixes
        find_library(FFTW2_MPI_R_LIB NAMES ${FFTW2_LIB_PREFIX_R}fftw_mpi HINTS ENV FFTW2_ROOT ${FFTW2_ROOT} PATH_SUFFIXES lib)
        find_library(FFTW2_MPI_LIB NAMES ${FFTW2_LIB_PREFIX}fftw_mpi HINTS ENV FFTW2_ROOT ${FFTW2_ROOT} PATH_SUFFIXES lib)
        find_library(FFTW2_CORE_R_LIB NAMES ${FFTW2_LIB_PREFIX_R}fftw HINTS ENV FFTW2_ROOT ${FFTW2_ROOT} PATH_SUFFIXES lib)
        find_library(FFTW2_CORE_LIB NAMES ${FFTW2_LIB_PREFIX}fftw HINTS ENV FFTW2_ROOT ${FFTW2_ROOT} PATH_SUFFIXES lib)

        if(NOT FFTW2_MPI_R_LIB OR NOT FFTW2_MPI_LIB OR NOT FFTW2_CORE_R_LIB OR NOT FFTW2_CORE_LIB)
             message(FATAL_ERROR "Could not find all required FFTW2 library components (${FFTW2_LIB_PREFIX_R}fftw_mpi, ${FFTW2_LIB_PREFIX}fftw_mpi, ${FFTW2_LIB_PREFIX_R}fftw, ${FFTW2_LIB_PREFIX}fftw). Check FFTW2 installation and paths (FFTW2_ROOT, CMAKE_PREFIX_PATH).")
        endif()

        set(FFTW_INCLUDE_DIRS ${FFTW2_INCLUDE_DIR})
        # Replicate order from original Makefile: -lrfftw_mpi -lfftw_mpi -lrfftw -lfftw (using found paths)
        set(FFTW_LIBRARIES ${FFTW2_MPI_R_LIB} ${FFTW2_MPI_LIB} ${FFTW2_CORE_R_LIB} ${FFTW2_CORE_LIB})
        message(STATUS "FFTW2 Include Dirs: ${FFTW_INCLUDE_DIRS}")
        message(STATUS "FFTW2 Libraries: ${FFTW_LIBRARIES}")
        # No USE_FFTW3 definition added here
    endif() # End of USE_FFTW3 check

    # Common steps if FFTW is needed (either version)
    set(FFTW_IS_ENABLED TRUE)
    list(APPEND COMPILE_DEFINITIONS_TO_ADD ${FFTW_COMPILE_DEF}) # Add DOUBLEPRECISION_FFTW if needed

else() # NEED_FFTW is OFF
    message(STATUS "FFTW library is not required.")
    set(FFTW_IS_ENABLED FALSE)
endif() # End of NEED_FFTW check

# --- CHIMES / Sundials (Conditional) ---
if(CHIMES)
    # Requires FindSundials.cmake or manual hints (SUNDIALS_ROOT / CMAKE_PREFIX_PATH)
    # Specify components needed by GIZMO (based on Makefile)
    find_package(Sundials REQUIRED COMPONENTS CVODE KINSOL NVECTORSERIAL)
    message(STATUS "Sundials Found: ${Sundials_FOUND}")
    message(STATUS "Sundials Include Dirs: ${Sundials_INCLUDE_DIRS}")
    message(STATUS "Sundials Libraries: ${Sundials_LIBRARIES}")
    set(CHIMES_IS_ENABLED TRUE)
    if(CHIMES_USE_DOUBLE_PRECISION)
        list(APPEND COMPILE_DEFINITIONS_TO_ADD CHIMES_USE_DOUBLE_PRECISION)
    endif()
else()
    set(CHIMES_IS_ENABLED FALSE)
endif()

# --- Grackle (Conditional) ---
if(COOL_GRACKLE)
    # Requires FindGrackle.cmake or manual hints (GRACKLE_ROOT / CMAKE_PREFIX_PATH)
    find_library(GRACKLE_LIBRARY NAMES grackle)
    find_path(GRACKLE_INCLUDE_DIR NAMES grackle.h PATH_SUFFIXES include)
    if(NOT GRACKLE_LIBRARY OR NOT GRACKLE_INCLUDE_DIR)
        message(FATAL_ERROR "Grackle library/include not found. Set GRACKLE_ROOT or add to CMAKE_PREFIX_PATH.")
    endif()
    message(STATUS "Grackle Found: YES")
    message(STATUS "Grackle Include Dir: ${GRACKLE_INCLUDE_DIR}")
    message(STATUS "Grackle Library: ${GRACKLE_LIBRARY}")
    set(GRACKLE_IS_ENABLED TRUE)
    list(APPEND COMPILE_DEFINITIONS_TO_ADD COOL_GRACKLE CONFIG_BFLOAT_8) # From Makefile
else()
    set(GRACKLE_IS_ENABLED FALSE)
endif()

# --- PThreads (Conditional) ---
if(PTHREADS_NUM_THREADS)
    find_package(Threads REQUIRED)
    set(PTHREADS_IS_ENABLED TRUE)
    list(APPEND COMPILE_DEFINITIONS_TO_ADD PTHREADS_NUM_THREADS)
else()
    set(PTHREADS_IS_ENABLED FALSE)
endif()

#--------------------------------------------------------------------------
# Source File Lists
#--------------------------------------------------------------------------
set(CORE_SOURCES
    main.c accel.c timestep.c init.c restart.c io.c
    predict.c global.c begrun.c run.c allvars.c read_ic.c
    domain.c driftfac.c kicks.c ngb.c # compile_time_info.c added later
    merge_split.c
)
set(SYSTEM_SOURCES
    system/system.c
    system/allocate.c
    system/mymalloc.c
    system/parallel_sort.c
    system/peano.c
    system/parallel_sort_special.c
    system/mpi_util.c
    system/pinning.c
)
set(GRAVITY_SOURCES
    gravity/forcetree.c
    gravity/forcetree_update.c
    gravity/gravtree.c
    gravity/cosmology.c
    gravity/potential.c
    gravity/ags_hsml.c
    gravity/binary.c
)
if(PMGRID)
    list(APPEND GRAVITY_SOURCES
         gravity/pm_periodic.c
         gravity/pm_nonperiodic.c # Often linked with PMGRID
         gravity/longrange.c      # Often linked with PMGRID
    )
endif()

set(HYDRO_SOURCES
    hydro/hydro_toplevel.c
    hydro/density.c
    hydro/gradients.c
)
if(TURB_DRIVING_SPECTRUMGRID)
    list(APPEND HYDRO_SOURCES
         turb/dynamic_diffusion.c
         turb/dynamic_diffusion_velocities.c
         turb/turb_driving.c
         turb/turb_powerspectra.c
    )
endif()

set(EOSCOOL_SOURCES
    cooling/cooling.c
    eos/eos.c
    eos/hydrogen_molecule.c
    eos/cosmic_ray_fluid/cosmic_ray_alfven.c
    eos/cosmic_ray_fluid/cosmic_ray_utilities.c
    solids/elastic_physics.c
    solids/grain_physics.c
    solids/ism_dust_chemistry.c
    nuclear/nuclear_network_solver.c
    nuclear/nuclear_network.c
)
set(STARFORM_SOURCES
    galaxy_sf/sfr_eff.c
    galaxy_sf/stellar_evolution.c
    galaxy_sf/mechanical_fb.c
    galaxy_sf/thermal_fb.c
    galaxy_sf/radfb_local.c
    galaxy_sf/dm_dispersion_hsml.c
)
set(SINK_SOURCES
    galaxy_sf/blackholes/blackhole.c
    galaxy_sf/blackholes/blackhole_util.c
    galaxy_sf/blackholes/blackhole_environment.c
    galaxy_sf/blackholes/blackhole_feed.c
    galaxy_sf/blackholes/blackhole_swallow_and_kick.c
)
set(RHD_SOURCES
    radiation/rt_utilities.c
    radiation/rt_CGmethod.c
    radiation/rt_source_injection.c
    radiation/rt_chem.c
    radiation/rt_cooling.c
)
set(FOF_SOURCES
    structure/fof.c
    structure/subfind/subfind.c
    structure/subfind/subfind_vars.c
    structure/subfind/subfind_collective.c
    structure/subfind/subfind_serial.c
    structure/subfind/subfind_so.c
    structure/subfind/subfind_cont.c
    structure/subfind/subfind_distribute.c
    structure/subfind/subfind_findlinkngb.c
    structure/subfind/subfind_nearesttwo.c
    structure/subfind/subfind_loctree.c
    structure/subfind/subfind_potential.c
    structure/subfind/subfind_density.c
    structure/twopoint.c
    structure/lineofsight.c
)
set(MISC_SOURCES
    sidm/cbe_integrator.c
    sidm/dm_fuzzy.c
    sidm/sidm_core.c
)

# Conditional Sources
set(ALL_FORTRAN_SOURCES "")
if(EOS_HELMHOLTZ)
    list(APPEND EOSCOOL_SOURCES eos/eos_interface.c)
    set(ALL_FORTRAN_SOURCES
        eos/helmholtz/helm_impl.f90
        eos/helmholtz/helm_wrap.f90
    )
    list(APPEND COMPILE_DEFINITIONS_TO_ADD EOS_HELMHOLTZ)
    message(STATUS "Helmholtz EOS Fortran sources enabled.")
endif()
if(CHIMES_IS_ENABLED)
    list(APPEND EOSCOOL_SOURCES
         cooling/chimes/chimes.c
         cooling/chimes/chimes_cooling.c
         cooling/chimes/init_chimes.c
         cooling/chimes/rate_equations.c
         cooling/chimes/update_rates.c)
    list(APPEND COMPILE_DEFINITIONS_TO_ADD CHIMES)
    message(STATUS "CHIMES sources enabled.")
endif()
if(GRACKLE_IS_ENABLED)
    list(APPEND EOSCOOL_SOURCES cooling/grackle.c)
    message(STATUS "Grackle source enabled.")
endif()

# Combine all C/C++ sources
set(ALL_C_CXX_SOURCES
    ${CORE_SOURCES}
    ${SYSTEM_SOURCES}
    ${GRAVITY_SOURCES}
    ${HYDRO_SOURCES}
    ${EOSCOOL_SOURCES}
    ${STARFORM_SOURCES}
    ${SINK_SOURCES}
    ${RHD_SOURCES}
    ${FOF_SOURCES}
    ${MISC_SOURCES}
)

# #--------------------------------------------------------------------------
# # Generate
# # * Configuration Header (GIZMO_config.h)
# # * Compile Time Info Source (compile_time_info.c)
# #--------------------------------------------------------------------------
# # 1. Find the Perl executable
# find_package(Perl REQUIRED)

# # 2. Define file paths (adjust if script/config are elsewhere)
# set(CONFIG_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/prepare-config.perl)
# set(CONFIG_FILE ${CMAKE_CURRENT_SOURCE_DIR}/Config.sh)

# # Define paths for the generated files in the build directory
# set(GENERATED_HEADER ${CMAKE_SOURCE_DIR}/GIZMO_config.h)
# set(GENERATED_SOURCE ${CMAKE_SOURCE_DIR}/compile_time_info.c)

# # 3. Define the custom command to run the Perl script
# add_custom_command(
#     OUTPUT ${GENERATED_HEADER} ${GENERATED_SOURCE}  # Files this command creates
#     COMMAND ${PERL_EXECUTABLE} ${CONFIG_SCRIPT} ${CONFIG_FILE} # The command to run
#     WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}     # Where to run it (so outputs go here)
#     DEPENDS ${CONFIG_SCRIPT} ${CONFIG_FILE}           # Re-run if script or Config.sh changes
#     COMMENT "Generating GIZMO configuration files from ${CONFIG_FILE}"
#     VERBATIM                                         # Pass command arguments literally
# )

#--------------------------------------------------------------------------
# Generate Configuration Header (GIZMO_config.h)
#--------------------------------------------------------------------------
# Requires a template file GIZMO_config.h.in in the source directory
# Example GIZMO_config.h.in:
#   #ifndef GIZMO_CONFIG_H
#   #define GIZMO_CONFIG_H
#   /* CMake generated options */
#   #cmakedefine OPENMP
#   #cmakedefine CHIMES
#   #cmakedefine EOS_HELMHOLTZ
#   /* ... add #cmakedefine for ALL options ... */
#   #cmakedefine HDF5_IS_ENABLED /* Define if HDF5 is used */
#   #endif
# configure_file(GIZMO_config.h.in ${CMAKE_BINARY_DIR}/GIZMO_config.h)

#--------------------------------------------------------------------------
# Generate Compile Time Info Source
#--------------------------------------------------------------------------
# Requires a template file compile_time_info.c.in in the source directory
# Example compile_time_info.c.in:
#   #include "allvars.h" // or appropriate header
#   #include "GIZMO_config.h" // Include the generated header
#   const char* BuildInfo = "@BUILDINFO_STRING@";
#   void output_compile_time_options(void) { /* Print options based on #ifdefs */ }
# configure_file(compile_time_info.c.in ${CMAKE_BINARY_DIR}/compile_time_info.c @ONLY) # @ONLY prevents replacing #cmakedefine
list(APPEND ALL_C_CXX_SOURCES ${CMAKE_SOURCE_DIR}/compile_time_info.c)

#--------------------------------------------------------------------------
# Executable Target Definition
#--------------------------------------------------------------------------
add_executable(GIZMO ${ALL_C_CXX_SOURCES} ${ALL_FORTRAN_SOURCES})
set_target_properties(GIZMO PROPERTIES CXX_STANDARD 11) # Example: Set C++ standard if needed
set_target_properties(GIZMO PROPERTIES C_STANDARD 99)   # Example: Set C standard if needed

#--------------------------------------------------------------------------
# Target Properties (Include Dirs, Compile Definitions, Link Libs)
#--------------------------------------------------------------------------
# --- Include Directories ---
target_include_directories(GIZMO PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}> # Headers in source dir
    $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}>         # Generated GIZMO_config.h
    ${MPI_INCLUDE_PATH}    # From find_package(MPI) - may need MPI::MPI_C includes instead
    ${GSL_INCLUDE_DIRS}    # From find_package(GSL)
)
if(FFTW_IS_ENABLED)
    target_include_directories(GIZMO PUBLIC ${FFTW_INCLUDE_DIRS})
endif()
if(HDF5_IS_ENABLED)
    target_include_directories(GIZMO PUBLIC ${HDF5_INCLUDE_DIRS})
endif()
if(CHIMES_IS_ENABLED)
    target_include_directories(GIZMO PUBLIC ${Sundials_INCLUDE_DIRS})
endif()
if(GRACKLE_IS_ENABLED)
    target_include_directories(GIZMO PUBLIC ${GRACKLE_INCLUDE_DIR})
endif()

# --- Compile Definitions ---
target_compile_definitions(GIZMO PRIVATE
    "BUILDINFO=\"${BUILDINFO_STRING}\"" # Pass build string (escaped)
    ${COMPILE_DEFINITIONS_TO_ADD}      # Add definitions accumulated earlier
)
# Add other definitions based on options
if(USE_MPI_IN_PLACE)
    target_compile_definitions(GIZMO PRIVATE USE_MPI_IN_PLACE)
endif()
if(NO_ISEND_IRECV_IN_DOMAIN)
    target_compile_definitions(GIZMO PRIVATE NO_ISEND_IRECV_IN_DOMAIN)
endif()
if(HDF5_IS_ENABLED AND HDF5_DISABLE_VERSION_CHECK)
    target_compile_definitions(GIZMO PRIVATE HDF5_DISABLE_VERSION_CHECK)
endif()
if(DISABLE_ALIGNED_ALLOC)
    target_compile_definitions(GIZMO PRIVATE DISABLE_ALIGNED_ALLOC)
endif()
if(REDUCE_TREEWALK_BRANCHING)
    target_compile_definitions(GIZMO PRIVATE REDUCE_TREEWALK_BRANCHING)
endif()
if(PMGRID)
    target_compile_definitions(GIZMO PRIVATE PMGRID)
endif()
if(TURB_DRIVING_SPECTRUMGRID)
    target_compile_definitions(GIZMO PRIVATE TURB_DRIVING_SPECTRUMGRID)
endif()
if(OLD_HDF5 AND HDF5_IS_ENABLED)
    target_compile_definitions(GIZMO PRIVATE OLD_HDF5)
endif()
if(HDF5_IS_ENABLED)
    target_compile_definitions(GIZMO PRIVATE HDF5_IS_ENABLED) # Useful for #ifdef HDF5_IS_ENABLED
    target_compile_definitions(GIZMO PRIVATE H5_USE_16_API)
endif()
# ... Add other definitions corresponding to Makefile OPT flags ...


# --- Base Compiler Flags (Warnings, etc.) ---
# Add flags common to all builds/systems. Optimization flags should come from toolchain/environment.
target_compile_options(GIZMO PRIVATE
    $<$<COMPILE_LANGUAGE:C>:-Wall>
    $<$<COMPILE_LANGUAGE:CXX>:-Wall>
    # Add other general flags as needed
)
# Example of standard specification (already set via target properties)
# target_compile_options(GIZMO PRIVATE $<$<COMPILE_LANGUAGE:C>:-std=c99>)
# target_compile_options(GIZMO PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-std=c++11>)


# --- Linking ---
target_link_libraries(GIZMO PUBLIC
    MPI::MPI_C       # Use CMake imported targets for MPI
    MPI::MPI_CXX
    ${GSL_LIBRARIES} # Use variable from find_package(GSL)
)
# Link math library (usually needed)
target_link_libraries(GIZMO PUBLIC m)

if(FFTW_IS_ENABLED)
    target_link_libraries(GIZMO PUBLIC ${FFTW_LIBRARIES})
endif()
if(HDF5_IS_ENABLED)
    target_link_libraries(GIZMO PUBLIC ${HDF5_LIBRARIES})
endif()
if(CHIMES_IS_ENABLED)
    target_link_libraries(GIZMO PUBLIC ${Sundials_LIBRARIES})
endif()
if(GRACKLE_IS_ENABLED)
    target_link_libraries(GIZMO PUBLIC ${GRACKLE_LIBRARY})
endif()
if(EOS_HELMHOLTZ)
    # Ensure Fortran MPI libraries are linked if Fortran is used
    target_link_libraries(GIZMO PUBLIC MPI::MPI_Fortran)
endif()
if(PTHREADS_IS_ENABLED)
    target_link_libraries(GIZMO PUBLIC Threads::Threads)
endif()

# --- OpenMP Handling ---
if(OPENMP)
    find_package(OpenMP REQUIRED)
    message(STATUS "OpenMP Found: YES")
    message(STATUS "OpenMP C Flags: ${OpenMP_C_FLAGS}")
    message(STATUS "OpenMP CXX Flags: ${OpenMP_CXX_FLAGS}")
    message(STATUS "OpenMP Fortran Flags: ${OpenMP_Fortran_FLAGS}") # If Fortran is used
    target_compile_options(GIZMO PUBLIC ${OpenMP_C_FLAGS})
    target_compile_options(GIZMO PUBLIC ${OpenMP_CXX_FLAGS})
    # Linker flags for OpenMP are often the same as compile flags
    target_link_libraries(GIZMO PUBLIC ${OpenMP_C_FLAGS})
    if(EOS_HELMHOLTZ)
         target_compile_options(GIZMO PUBLIC ${OpenMP_Fortran_FLAGS})
         target_link_libraries(GIZMO PUBLIC ${OpenMP_Fortran_FLAGS})
    endif()
    # Define OPENMP macro for the code
    target_compile_definitions(GIZMO PRIVATE OPENMP)
else()
     message(STATUS "OpenMP support disabled.")
endif()


#--------------------------------------------------------------------------
# Installation (Optional)
#--------------------------------------------------------------------------
# include(GNUInstallDirs)
# install(TARGETS GIZMO RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
# install(FILES ${CMAKE_BINARY_DIR}/GIZMO_config.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})


message(STATUS "GIZMO configuration complete. Use 'make' or your chosen generator to build.")