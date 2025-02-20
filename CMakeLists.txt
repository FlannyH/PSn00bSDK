# PSn00bSDK main build script
# (C) 2021-2022 spicyjpeg - MPL licensed

# NOTE: CMake doesn't support using multiple toolchains in a single project,
# so we can't use add_subdirectory() to build both the libraries and tools. A
# workaround is to use ExternalProject_Add() to launch multiple independent
# CMake instances, creating what's known as a "superbuild".

cmake_minimum_required(VERSION 3.21)
include(ExternalProject)

project(
	PSn00bSDK
	LANGUAGES    NONE
	VERSION      0.23
	DESCRIPTION  "Open source PlayStation 1 SDK"
	HOMEPAGE_URL "http://lameguy64.net/?page=psn00bsdk"
)

# Including this without initializing at least one language throws a warning and
# there's no way to mute it.
include(GNUInstallDirs)

## Settings

# These are passed through to libpsn00b and the examples (they are defined in
# the toolchain file).
set(
	PSN00BSDK_TC ""
	CACHE PATH   "Path to the GCC toolchain's installation directory (if not in PATH)"
)
set(
	PSN00BSDK_TARGET mipsel-none-elf
	CACHE STRING     "GCC toolchain target triplet"
)

set(
	MKPSXISO_NO_LIBFLAC OFF
	CACHE BOOL          "Disable libflac integration when building mkpsxiso"
)
set(
	SKIP_EXAMPLES OFF
	CACHE BOOL    "Skip building SDK examples (not required for installation)"
)
set(
	LIBPSN00B_GENERATOR ${CMAKE_GENERATOR}
	CACHE STRING        "CMake generator to use for building libpsn00b and examples"
)
set(
	PSN00BSDK_GIT_TAG ""
	CACHE STRING      "Git tag or branch name (used by CI)"
)
set(
	PSN00BSDK_GIT_COMMIT ""
	CACHE STRING         "Git commit hash (used by CI)"
)

# Attempt to automatically select a suitable CMake generator to build libpsn00b
# and the examples. Only Ninja and makefile-based generators can be used, as
# other generators (VS and Xcode) do not allow custom toolchains to be used.
if(NOT LIBPSN00B_GENERATOR MATCHES ".*(Make|Makefiles|Ninja)( Multi-Config)?$")
	find_program(_ninja ninja NO_CACHE)

	if(_ninja STREQUAL "_ninja-NOTFOUND")
		message(FATAL_ERROR "LIBPSN00B_GENERATOR must be set to a CMake generator which supports custom toolchains (Ninja or make). The current default generator (${CMAKE_GENERATOR}) will be used to build tools.")
	else()
		set(
			LIBPSN00B_GENERATOR Ninja
			CACHE STRING        "CMake generator to use for building libpsn00b and examples"
			FORCE
		)
		message(WARNING "The current default generator (${CMAKE_GENERATOR}) will be used to build tools but does not support custom toolchains. Ninja will be used instead for building libpsn00b and examples.")
	endif()
endif()

string(TIMESTAMP PSN00BSDK_BUILD_DATE UTC)

# Forward some important variables to mkpsxiso and to the subprojects (they are
# not inherited automatically as they are not environment variables). This also
# sets all subprojects to "install" everything to a temporary directory in the
# build tree, so they don't actually get installed until "cmake --install" is
# invoked (ExternalProject_Add() runs the subprojects' install step at build
# time).
set(
	_common_args
	-DPSN00BSDK_TC:PATH=${PSN00BSDK_TC}
	-DPSN00BSDK_TARGET:STRING=${PSN00BSDK_TARGET}
	-DPSN00BSDK_VERSION:STRING=${PROJECT_VERSION}
	-DPSN00BSDK_BUILD_DATE:STRING=${PSN00BSDK_BUILD_DATE}
	-DPSN00BSDK_GIT_TAG:STRING=${PSN00BSDK_GIT_TAG}
	-DPSN00BSDK_GIT_COMMIT:STRING=${PSN00BSDK_GIT_COMMIT}
)
set(
	_tools_args
	${_common_args}
	-DCMAKE_TOOLCHAIN_FILE:FILEPATH=${CMAKE_TOOLCHAIN_FILE}
	-DCMAKE_INSTALL_PREFIX:PATH=${PROJECT_BINARY_DIR}/tree
	-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
	-DMKPSXISO_NO_LIBFLAC:BOOL=${MKPSXISO_NO_LIBFLAC}
)
set(
	_libpsn00b_args
	${_common_args}
	-DCMAKE_TOOLCHAIN_FILE:FILEPATH=${CMAKE_TOOLCHAIN_FILE}
	-DCMAKE_INSTALL_PREFIX:PATH=${PROJECT_BINARY_DIR}/tree
)
set(
	_examples_args
	${_common_args}
	-DCMAKE_TOOLCHAIN_FILE:FILEPATH=${PROJECT_BINARY_DIR}/tree/${CMAKE_INSTALL_LIBDIR}/libpsn00b/cmake/sdk.cmake
	-DCMAKE_INSTALL_PREFIX:PATH=${PROJECT_BINARY_DIR}/examples
	-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
)

# Ensure PSn00bSDK isn't being built using the toolchain file from PSn00bSDK
# itself (or from another version of it).
if(CMAKE_TOOLCHAIN_FILE MATCHES ".*libpsn00b[/\\]cmake[/\\]sdk\.cmake$")
	unset(CMAKE_TOOLCHAIN_FILE CACHE)
	message(WARNING "CMAKE_TOOLCHAIN_FILE is currently set to the toolchain file of an existing PSn00bSDK installation. It will be ignored.")
endif()

## Subprojects

if(NOT EXISTS ${PROJECT_SOURCE_DIR}/tools/mkpsxiso/CMakeLists.txt)
	message(FATAL_ERROR "The mkpsxiso directory is empty. Run 'git submodule update --init --recursive' to populate it.")
endif()

ExternalProject_Add(
	tools
	PREFIX           subprojects
	SOURCE_DIR       ${PROJECT_SOURCE_DIR}/tools
	CMAKE_CACHE_ARGS ${_tools_args}
	INSTALL_DIR      tree
)
ExternalProject_Add(
	mkpsxiso
	PREFIX           subprojects
	SOURCE_DIR       ${PROJECT_SOURCE_DIR}/tools/mkpsxiso
	CMAKE_CACHE_ARGS ${_tools_args}
	INSTALL_DIR      tree
)
ExternalProject_Add(
	libpsn00b-debug
	PREFIX           subprojects
	SOURCE_DIR       ${PROJECT_SOURCE_DIR}/libpsn00b
	CMAKE_GENERATOR  ${LIBPSN00B_GENERATOR}
	CMAKE_CACHE_ARGS ${_libpsn00b_args} -DCMAKE_BUILD_TYPE:STRING=Debug
	INSTALL_DIR      tree
)
ExternalProject_Add(
	libpsn00b-release
	PREFIX           subprojects
	SOURCE_DIR       ${PROJECT_SOURCE_DIR}/libpsn00b
	CMAKE_GENERATOR  ${LIBPSN00B_GENERATOR}
	CMAKE_CACHE_ARGS ${_libpsn00b_args} -DCMAKE_BUILD_TYPE:STRING=Release
	INSTALL_DIR      tree
)
ExternalProject_Add(
	examples
	PREFIX           subprojects
	SOURCE_DIR       ${PROJECT_SOURCE_DIR}/examples
	CMAKE_GENERATOR  ${LIBPSN00B_GENERATOR}
	CMAKE_CACHE_ARGS ${_examples_args}
	INSTALL_DIR      examples
	DEPENDS          libpsn00b-debug libpsn00b-release tools mkpsxiso
	EXCLUDE_FROM_ALL ${SKIP_EXAMPLES}
)

# Install all files in the temporary installation tree, as well as static files
# from the source tree, when "cmake --install" is invoked.
foreach(
	_subdir IN ITEMS
		${CMAKE_INSTALL_BINDIR}
		${CMAKE_INSTALL_LIBDIR}
		${CMAKE_INSTALL_INCLUDEDIR}
		${CMAKE_INSTALL_DATADIR}
)
	install(
		# THE TRAILING SLASH IS IMPORTANT
		DIRECTORY   ${PROJECT_BINARY_DIR}/tree/${_subdir}/
		DESTINATION ${_subdir}
		COMPONENT   sdk
		USE_SOURCE_PERMISSIONS
	)
endforeach()

install(
	FILES       README.md LICENSE.md
	DESTINATION ${CMAKE_INSTALL_DATADIR}/psn00bsdk/doc
	COMPONENT   docs
)
install(
	DIRECTORY   doc template
	DESTINATION ${CMAKE_INSTALL_DATADIR}/psn00bsdk
	COMPONENT   docs
)
install(
	DIRECTORY   examples
	DESTINATION ${CMAKE_INSTALL_DATADIR}/psn00bsdk
	COMPONENT   examples
)

## CPack configuration

include(cpack/setup.cmake)
