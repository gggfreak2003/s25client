
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.12)
IF(NOT CMAKE_VERSION VERSION_LESS 3.0)
	# RPath enabled by default. Rpath only supported in CMake >2.8.12
	cmake_policy(SET CMP0042 OLD)
ENDIF()
IF(NOT CMAKE_VERSION VERSION_LESS 3.1)
	# Simplified variable expansion. No change for usual variable references so use faster one
	cmake_policy(SET CMP0053 NEW)
	# Only interpret if() arguments as variables or keywords when unquoted. 
	cmake_policy(SET CMP0054 NEW)
ENDIF()

#Check for ccache
find_program(CCACHE_PROGRAM ccache)
MARK_AS_ADVANCED(CCACHE_PROGRAM)
if(CCACHE_PROGRAM)
	MESSAGE(STATUS "Using ccache to speed up builds")
	set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

PROJECT(s25client)

################################################################################

message(STATUS "Using CMake ${CMAKE_VERSION}")

IF (DEFINED CMAKE_TOOLCHAIN_FILE)
	MESSAGE(STATUS "Used Toolchain definition file '${CMAKE_TOOLCHAIN_FILE}'")
ENDIF()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules" "${CMAKE_SOURCE_DIR}/libutil/cmake/Modules")

# Just include required
INCLUDE(CMakeMacroEnsureOutOfSourceBuild)

include(CheckStructHasMember)
INCLUDE(CheckIncludeFile)

################################################################################
# Include platform specific config files
################################################################################

STRING(TOLOWER ${CMAKE_SYSTEM_NAME} LOWER_SYSTEM_NAME)
IF (CMAKE_CROSSCOMPILING)
	MESSAGE(STATUS "Configuring for cross-compiling to ${CMAKE_SYSTEM} on ${CMAKE_SYSTEM_PROCESSOR}")
	#Include cross-compile platform config
    set(_config "cmake/c.${LOWER_SYSTEM_NAME}.cmake")
	IF (EXISTS "${CMAKE_CURRENT_LIST_DIR}/${_config}")
		MESSAGE(STATUS "Using cross-platform config ${_config}")
		INCLUDE("${_config}")
	ENDIF()
ELSE()
	MESSAGE(STATUS "Configuring for native compiling to ${CMAKE_SYSTEM} on ${CMAKE_SYSTEM_PROCESSOR}")
ENDIF()

#Include generic platform config
set(_config "cmake/${LOWER_SYSTEM_NAME}.cmake")
IF (EXISTS "${CMAKE_CURRENT_LIST_DIR}/${_config}")
    MESSAGE(STATUS "Using platform config ${_config}")
    INCLUDE("${_config}")
ENDIF()

OPTION(RTTR_ENABLE_OPTIMIZATIONS "Build with optimizing flags (such as -O2 and -ffast-math added to CFLAGS and CXXFLAGS)" ON)
if(RTTR_ENABLE_OPTIMIZATIONS)
    include("cmake/optimizations.cmake")
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -ggdb")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -ggdb")
endif()

################################################################################
# Add contrib libraries. Might need different compile flags
# Need to use include() so we can set and add variables/libraries to current scope
include("${CMAKE_CURRENT_SOURCE_DIR}/contrib.cmake")

################################################################################
# ClangFormat
################################################################################

find_package(ClangFormat 5.0.0)

################################################################################
# Set paths.
# Note: Mixing relative and absolute paths is not recommended!
# If RTTR_BINDIR is absolute then relative paths are relative to CMAKE_INSTALL_PREFIX
# Else a prefix path is calculated based on RTTR_BINDIR and the current executable path so that we have <PrefixPath>/RTTR_BINDIR
# In this case all relative paths are relative to that prefix path
# Example: RTTR_BINDIR="bin", RTTR_DATADIR="data" executed in "/foo/bar/bin/rttr.exe" -> FULL_DATADIR=/foo/bar/data
################################################################################

IF (WIN32)
	# Windows uses one directory only
	SET(RTTR_BINDIR "." CACHE INTERNAL "")
	SET(RTTR_DATADIR "." CACHE INTERNAL "")
	SET(RTTR_GAMEDIR "." CACHE INTERNAL "")
	SET(RTTR_LIBDIR "." CACHE INTERNAL "")
	SET(RTTR_DOCDIR "." CACHE INTERNAL "")
	SET(RTTR_EXTRA_BINDIR "./RTTR" CACHE INTERNAL "")
	SET(RTTR_DRIVERDIR "./driver" CACHE INTERNAL "")
ELSE()
	include(GNUInstallDirs)
	SET(RTTR_BINDIR "${CMAKE_INSTALL_BINDIR}" CACHE STRING "Directory for the binaries")
	SET(RTTR_DATADIR "${CMAKE_INSTALL_DATAROOTDIR}/s25rttr" CACHE STRING "Data directory")
	if("${RTTR_DATADIR}" STREQUAL "")
		# Avoid using absolute path accidentally
		SET(DEFAULT_GAMEDIR "S2")
	else()
		SET(DEFAULT_GAMEDIR "${RTTR_DATADIR}/S2")
	endif()
	SET(RTTR_GAMEDIR "${DEFAULT_GAMEDIR}" CACHE STRING "(Original) Settlers 2 data directory")
	SET(RTTR_LIBDIR "${CMAKE_INSTALL_LIBDIR}" CACHE STRING "Directory for shared binaries")
	# The default contains PROJECTNAME which would be "s25client"
	string(REPLACE "${PROJECT_NAME}" "s25rttr" DEFAULT_DOCDIR "${CMAKE_INSTALL_DOCDIR}")
	SET(RTTR_DOCDIR "${DEFAULT_DOCDIR}" CACHE STRING "Directory for documentation files")
	if("${RTTR_LIBDIR}" STREQUAL "")
		# Avoid using absolute path accidentally
		SET(DEFAULT_DRIVERDIR "driver")
	else()
		SET(DEFAULT_DRIVERDIR "${RTTR_LIBDIR}/driver")
	endif()
	SET(RTTR_EXTRA_BINDIR "${CMAKE_INSTALL_LIBEXECDIR}/s25rttr" CACHE STRING "Path to binaries used internally")
	SET(RTTR_DRIVERDIR "${DEFAULT_DRIVERDIR}" CACHE STRING "Directory for driver objects")
ENDIF()

SET(RTTR_SRCDIR ${CMAKE_CURRENT_SOURCE_DIR} CACHE INTERNAL "Path to source files. Used for shell scripts and configure")
set(RTTR_S2_PLACEHOLDER_PATH "${CMAKE_BINARY_DIR}/${RTTR_GAMEDIR}/put your S2-Installation in here")
set(RTTR_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX} CACHE INTERNAL "Used for configure scripts")

# Output directories
set(RTTR_OUTPUT_DIR ${CMAKE_BINARY_DIR})
if(CMAKE_CONFIGURATION_TYPES)
    foreach(type IN LISTS CMAKE_CONFIGURATION_TYPES)
        string(TOUPPER ${type} _UPPER_TYPE)
        set(RTTR_OUTPUT_DIR_${_UPPER_TYPE} ${RTTR_OUTPUT_DIR}/${type})
    endforeach()
elseif(CMAKE_BUILD_TYPE)
    string(TOUPPER ${CMAKE_BUILD_TYPE} _UPPER_TYPE)
    set(RTTR_OUTPUT_DIR_${_UPPER_TYPE} ${RTTR_OUTPUT_DIR})
endif()

# Helper function to set output dirs for all configurations
# output_type is RUNTIME, LIBRARY, ...
# subDir is the subdirectory in RTTR_OUTPUT_DIR_XXX
macro(rttr_set_output_dir output_type subDir)
    if(CMAKE_CONFIGURATION_TYPES)
        foreach(type IN LISTS CMAKE_CONFIGURATION_TYPES)
            string(TOUPPER ${type} _UPPER_TYPE)
            set(CMAKE_${output_type}_OUTPUT_DIRECTORY_${_UPPER_TYPE} ${RTTR_OUTPUT_DIR_${_UPPER_TYPE}}/${subDir})
        endforeach()
    elseif(CMAKE_BUILD_TYPE)
        string(TOUPPER ${CMAKE_BUILD_TYPE} _UPPER_TYPE)
        set(CMAKE_${output_type}_OUTPUT_DIRECTORY_${_UPPER_TYPE} ${RTTR_OUTPUT_DIR}/${subDir})
    else()
        set(CMAKE_${output_type}_OUTPUT_DIRECTORY ${RTTR_OUTPUT_DIR}/${subDir})
    endif()
endmacro()
rttr_set_output_dir(RUNTIME ${RTTR_BINDIR})
rttr_set_output_dir(LIBRARY ${RTTR_LIBDIR})

################################################################################
# Definitions and flags
################################################################################

# Do this before adding -Werror!
check_struct_has_member("struct timespec" tv_sec time.h HAVE_STRUCT_TIMESPEC)
ADD_DEFINITIONS(-DBOOST_PP_VARIADICS=1)

CHECK_INCLUDE_FILE(valgrind/memcheck.h HAVE_MEMCHECK_H)
IF(HAVE_MEMCHECK_H)
	ADD_DEFINITIONS(-DHAVE_MEMCHECK_H)
ENDIF()

# Clang bug workaround
if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	try_compile(CHECK_CLANG_INLINE "${CMAKE_CURRENT_BINARY_DIR}/cxx" "${CMAKE_SOURCE_DIR}/cmake/checkclang.cpp")
	if(NOT ${CHECK_CLANG_INLINE})
		add_definitions("-D__extern_always_inline=extern __always_inline __attribute__ ((__gnu_inline__))")
	endif()

	# Some workaround flags for execssive warnings in clang
	CheckAndAddFlag(-Wno-unused-local-typedefs)
endif()

# Enable warnings
include(cmake/warnings.cmake)

################################################################################
# Code coverage
################################################################################

# VS does not support coverage analysis
if(NOT MSVC)
	option(RTTR_ENABLE_COVERAGE "Generate coverage build" OFF)
	if(RTTR_ENABLE_COVERAGE)
		if(NOT "${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
			message(FATAL_ERROR "Coverage requires a debug build or you get false positives")
		endif()
		# Note: "--coverage" instead of "-coverage" makes this work with ccache
		add_compile_options(-O0 --coverage)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
		if(CMAKE_COMPILER_IS_GNUCXX)
			# Inlining makes the coverage statistic much harder to read and may lead to lots of partials
			# However expect a slowdown!
			message(STATUS "Enabled no-inline for better coverage statistics")
			add_compile_options(-fno-inline -fno-inline-small-functions -fno-default-inline)
		else()
			message(STATUS "Using default inlining which probably influences coverage statistics")
		endif()
	endif()
endif()

################################################################################
# Configure files
################################################################################

IF (NOT WIN32)
    include(ConfigureExecutable)
	configure_executable("${CMAKE_CURRENT_SOURCE_DIR}/start.sh.cmake" . start.sh)
ENDIF()

file(COPY "bugle" DESTINATION ".")
file(WRITE "${RTTR_S2_PLACEHOLDER_PATH}" "put your S2-Installation in here")

enable_testing()

# Add dlls from contrib folder
if(MSVC)
	if(NOT EXISTS "${CONTRIB_DLL_DIR}/libcurl.dll")
		MESSAGE(WARNING "Folder with DLLs not found in ${CONTRIB_DLL_DIR}. You may not be able to execute directly from VS")
	else()
		FILE(GLOB CONTRIB_DLLS ${CONTRIB_DLL_DIR}/*.dll)
		include(GatherDll)
		gather_dll_add(${CONTRIB_DLLS})
	ENDIF()
endif()

################################################################################
# Include projects
################################################################################

# Building the updater when using non-default paths (especially absolute paths)
# is not recommended as it most likely won't work
set(RTTR_BUILD_UPDATER_DEF ON)
foreach(curDirVar BINDIR DATADIR LIBDIR DOCDIR DRIVERDIR)
	if(IS_ABSOLUTE ${RTTR_${curDirVar}})
		set(RTTR_BUILD_UPDATER_DEF OFF)
	endif()
endforeach()
option(RTTR_BUILD_UPDATER "Build auto-updater. Not advised when changing the default paths as the updater is configured specifically for the official builds." ${RTTR_BUILD_UPDATER_DEF})
################################################################################
# Set variables used by the release scripts and the updater:
#   PLATFORM_NAME in lowercase (windows, linux,...)
#   PLATFORM_ARCH: i386, x86_64, universal
################################################################################

STRING(TOLOWER ${CMAKE_SYSTEM_NAME} PLATFORM_NAME)

# Set PLATFORM_ARCH
IF(WIN32)
	IF (CMAKE_SIZEOF_VOID_P EQUAL 8)
		SET(PLATFORM_ARCH "x86_64")
	ELSEIF (CMAKE_SIZEOF_VOID_P EQUAL 4)
		SET(PLATFORM_ARCH "i386")
	ELSE()
		MESSAGE(FATAL_ERROR "Unknown platform for sizeof(void*)=${CMAKE_SIZEOF_VOID_P}")
	ENDIF ()
ELSE()
	if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "i.86")
		SET(PLATFORM_ARCH "i386")
	elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "amd64|AMD64|x86_64")
		SET(PLATFORM_ARCH "x86_64")
	ELSEIF(("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "universal" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "") AND APPLE)
		SET(PLATFORM_ARCH "universal")
	ELSE()
		MESSAGE(STATUS "Unofficial build with processor arch: ${CMAKE_SYSTEM_PROCESSOR}. Skipping release scripts and updater.")
        unset(PLATFORM_ARCH)
        set(RTTR_BUILD_UPDATER OFF CACHE INTERNAL "Disabled")
	ENDIF()
ENDIF()

ADD_SUBDIRECTORY(libutil)
ADD_SUBDIRECTORY(libendian)
ADD_SUBDIRECTORY(mygettext)
ADD_SUBDIRECTORY(libsiedler2)
# Build lobby client
SET(RTTR_BUILD_LOBBY_CLIENT TRUE)
ADD_SUBDIRECTORY(liblobby)
ADD_SUBDIRECTORY(s-c)
if(RTTR_BUILD_UPDATER)
	ADD_SUBDIRECTORY(s25update)
endif()

add_subdirectory(common)
ADD_SUBDIRECTORY(driver)
ADD_SUBDIRECTORY(rttrConfig)
ADD_SUBDIRECTORY(src)

option(RTTR_BUILD_EDITOR "Build editor." ON)
if(RTTR_BUILD_EDITOR)
	add_subdirectory(s25edit)
endif()

if(ClangFormat_FOUND)
	add_ClangFormat_folder(${CMAKE_CURRENT_SOURCE_DIR}/driver true)
	add_ClangFormat_folder(${CMAKE_CURRENT_SOURCE_DIR}/src true)
	add_ClangFormat_target("file")
endif()

################################################################################
# Install
################################################################################

# Start script
IF (WIN32)
	install(PROGRAMS "release/bin/rttr.bat" DESTINATION "${RTTR_BINDIR}")
ELSEIF(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "FreeBSD" OR CYGWIN)
	install(PROGRAMS "release/bin/rttr.sh" DESTINATION "${RTTR_BINDIR}")
ELSEIF(NOT APPLE)
	message(FATAL_ERROR "${CMAKE_SYSTEM_NAME} not supported")
ENDIF()

# RTTR directory, but exclude language input files
install(DIRECTORY "RTTR" DESTINATION "${RTTR_DATADIR}"
		PATTERN "*.po" EXCLUDE
		PATTERN "*.pot" EXCLUDE
		PATTERN ".*" EXCLUDE
)

# Documentation. TODO: Avoid recursive entries?
IF (NOT WIN32)
	install(DIRECTORY "RTTR/texte/" DESTINATION "${RTTR_DOCDIR}"
			FILES_MATCHING PATTERN "*.txt"
	)
ENDIF()

# Placeholder for S2 installation
install(FILES "${RTTR_S2_PLACEHOLDER_PATH}" DESTINATION "${RTTR_GAMEDIR}")

################################################################################
# Custom targets
################################################################################

IF(NOT CMAKE_HOST_WIN32 AND PLATFORM_ARCH)
    include(ConfigureExecutable)
    configure_executable("${CMAKE_CURRENT_SOURCE_DIR}/release/prepareRelease.sh.cmake" . prepareRelease.sh)
    configure_executable("${CMAKE_CURRENT_SOURCE_DIR}/release/create_release.sh.cmake" . create_release.sh)

	# Create nightly release
	ADD_CUSTOM_TARGET(create_nightly
					  COMMAND "${CMAKE_CURRENT_BINARY_DIR}/create_release.sh" nightly
					  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
	)

	# Create stable release
	ADD_CUSTOM_TARGET(create_stable
					  COMMAND "${CMAKE_CURRENT_BINARY_DIR_DIR}/create_release.sh" stable
					  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
	)
ENDIF()
