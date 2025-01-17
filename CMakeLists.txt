# Copyright (C) 2021 Michael Fabian Dirks
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <https://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.10)

# Detect if we are building by ourselves or as part of something else.
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_LIST_DIR}")
	set(GROUPED OFF)
	set(PREFIX "")
else()
	set(GROUPED ON)
	set(PREFIX "XUProfiler_")
endif()
set(LOGPREFIX "XUP: ")

################################################################################
# Options
################################################################################
set(${PREFIX}BUILD_EXAMPLES ON CACHE BOOL "Build Examples")
set(${PREFIX}BUILD_TESTS ON CACHE BOOL "Build Tests")
set(${PREFIX}DISABLE_NOINLINE OFF CACHE BOOL "Disable force-exlining code in supported compilers.")
set(${PREFIX}ENABLE_FORCEINLINE ON CACHE BOOL "Enable force-inlining code in supported compilers.")

################################################################################
# Versioning
################################################################################

set(VERSION_MAJOR 0)
set(VERSION_MINOR 0)
set(VERSION_PATCH 0)
set(VERSION_TWEAK 0)
set(VERSION_SUFFIX "")
set(VERSION_COMMIT "00000000")

# Check if we are in a git repository.
if(EXISTS "${CMAKE_CURRENT_LIST_DIR}/.git")
	# Try and figure out where git is.
	find_program(GIT git
		PATHS
			/bin
			/sbin
			/usr/bin
			/usr/local/bin
	)

	if(GIT)
		set(GIT_RESULT)
		set(GIT_OUTPUT)
		set(GIT_ERROR)

		execute_process(
			COMMAND "${GIT}" describe --tags --long --match "[0-9]*.[0-9]*.[0-9]*" --abbrev=8 HEAD
			WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
			RESULT_VARIABLE GIT_RESULT
			OUTPUT_VARIABLE GIT_OUTPUT
			ERROR_VARIABLE GIT_ERROR
			OUTPUT_STRIP_TRAILING_WHITESPACE
			ERROR_STRIP_TRAILING_WHITESPACE
			ERROR_QUIET
		)

		if(GIT_RESULT EQUAL 0)
			string(REPLACE "-" "." GIT_OUTPUT "${GIT_OUTPUT}")
			string(REPLACE "." ";" GIT_OUTPUT "${GIT_OUTPUT}")

			# Parse Version
			list(GET GIT_OUTPUT 0 VERSION_MAJOR)
			list(GET GIT_OUTPUT 1 VERSION_MINOR)
			list(GET GIT_OUTPUT 2 VERSION_PATCH)
			list(GET GIT_OUTPUT 3 VERSION_TWEAK)
			list(GET GIT_OUTPUT 4 VERSION_COMMIT)

			# Patch needs additional parsing.
			# This may be a [0-9]*[a-z]*[0-9]+ string.
			string(REGEX MATCHALL "^([0-9]+)([a-z]+[0-9]+)?" T_MATCHES "${VERSION_PATCH}")
			set(VERSION_PATCH "${CMAKE_MATCH_1}")
			if(CMAKE_MATCH_2)
				set(VERSION_SUFFIX "${CMAKE_MATCH_2}")
			else()
				set(VERSION_SUFFIX "")
			endif()
		else()
			message(WARNING "${LOGPREFIX}Failed to detect version, using default instead.")
		endif()
	endif()
else()
	message(STATUS "${LOGPREFIX}Not a git repository, automatic version detection disabled.")
endif()

# Allow manual overrides of the detected version.
set(${PREFIX}VERSION "" CACHE STRING "Override StreamFX version with this string. Format: Major.Minor.Patch[Suffix][-Tweak[-Commit8c]]")
if(NOT (${PREFIX}VERSION STREQUAL ""))
	string(REPLACE "-" "." T_VERSION "${${PREFIX}VERSION}")
	string(REPLACE "." ";" T_VERSION "${${PREFIX}VERSION}")

	list(LENGTH T_VERSION T_VERSIONLEN)
	list(GET T_VERSION 0 VERSION_MAJOR)
	list(GET T_VERSION 1 VERSION_MINOR)
	list(GET T_VERSION 2 VERSION_PATCH)
	if (T_VERSIONLEN GREATER_EQUAL 3)
		list(GET T_VERSION 3 VERSION_TWEAK)
	else()
		set(VERSION_BUILD 0)
	endif()
	if (T_VERSIONLEN GREATER_EQUAL 4)
		list(GET T_VERSION 4 VERSION_COMMIT)
	else()
		set(VERSION_COMMIT "")
	endif()

	# Patch needs additional parsing.
	# This may be a [0-9]*[a-z]*[0-9]+ string.
	string(REGEX MATCHALL "^([0-9]+)([a-z]+[0-9]+)?" T_MATCHES "${VERSION_PATCH}")
	set(VERSION_PATCH "${CMAKE_MATCH_1}")
	if(CMAKE_MATCH_2)
		set(VERSION_SUFFIX "${CMAKE_MATCH_2}")
	else()
		set(VERSION_SUFFIX "")
	endif()
endif()

# Generate Version String
if(NOT (VERSION_COMMIT STREQUAL ""))
	set(VERSION_STRING "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}.${VERSION_TWEAK}${VERSION_SUFFIX}-${VERSION_COMMIT}")
else()
	set(VERSION_STRING "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}.${VERSION_TWEAK}${VERSION_SUFFIX}")
endif()
message(STATUS "${LOGPREFIX}Version ${VERSION_STRING}")

################################################################################
# Define Project
################################################################################
project(
	xmr_utility_profiler
	VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}.${VERSION_TWEAK}
	DESCRIPTION "Simple but effective way to profile code."
	HOMEPAGE_URL "https://xaymar.com/"
)

################################################################################
# Define Library
################################################################################
set(PROJECT_SOURCES
	"source/xmr/utility/profiler/profiler.cpp"
	"source/xmr/utility/profiler/clock/hpc.cpp"
	"source/xmr/utility/profiler/clock/tsc.cpp"
)
set(PROJECT_HEADERS
	"include/xmr/utility/profiler/profiler.hpp"
	"include/xmr/utility/profiler/clock/hpc.hpp"
	"include/xmr/utility/profiler/clock/tsc.hpp"
)
set(PROJECT_TEMPLATES
	"templates/config.hpp.in"
)
set(PROJECT_GENERATED
)

# Generate config.hpp {
	if (${${PREFIX}ENABLE_FORCEINLINE})
		set(XMR_UTILITY_PROFILER_ENABLE_FORCEINLINE 1)
	endif()
	if (${${PREFIX}DISABLE_NOINLINE})
		set(XMR_UTILITY_PROFILER_DISABLE_NOINLINE 1)
	endif()
	if (${BUILD_SHARED_LIBS})
		set(XMR_UTILITY_PROFILER_SHARED_LIBRARY 1)
	endif()
	configure_file(
		"templates/config.hpp.in"
		"generated/xmr/utility/profiler/config.hpp"
		@ONLY
	)
	list(APPEND PROJECT_GENERATED
		"${PROJECT_BINARY_DIR}/generated/xmr/utility/profiler/config.hpp"
	)
# }

add_library(${PROJECT_NAME}
	${PROJECT_SOURCES}
	${PROJECT_HEADERS}
	${PROJECT_TEMPLATES}
	${PROJECT_GENERATED}
)

source_group(TREE "${PROJECT_SOURCE_DIR}/source" PREFIX "Sources" FILES ${PROJECT_SOURCES})
source_group(TREE "${PROJECT_SOURCE_DIR}/include" PREFIX "Headers" FILES ${PROJECT_HEADERS})
source_group(TREE "${PROJECT_SOURCE_DIR}/templates" PREFIX "Templates" FILES ${PROJECT_TEMPLATES})
source_group(TREE "${PROJECT_BINARY_DIR}/generated" PREFIX "Generated" FILES ${PROJECT_GENERATED})

set_target_properties(${PROJECT_NAME} PROPERTIES
	CXX_STANDARD 11
	CXX_STANDARD_REQUIRED ON
	CXX_EXTENSIONS OFF
)

target_include_directories(${PROJECT_NAME}
	PRIVATE
		"${PROJECT_SOURCE_DIR}/source"
		"${PROJECT_SOURCE_DIR}/include"
		"${PROJECT_BINARY_DIR}/generated"
	INTERFACE
		"$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
		"$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/generated>"
		"$<INSTALL_INTERFACE:include>"
)

target_link_libraries(${PROJECT_NAME}
	INTERFACE
)

if (BUILD_SHARED_LIBS)
	target_compile_definitions(${PROJECT_NAME}
		PRIVATE
			XMR_UTILITY_PROFILER_DO_LIBRARY_EXPORT
		INTERFACE
	)
endif()

################################################################################
# Install Library
################################################################################

# Binaries
install(
	TARGETS ${PROJECT_NAME}
	EXPORT xmr::utility::profiler
	ARCHIVE LIBRARY RUNTIME FRAMEWORK BUNDLE
	PERMISSIONS WORLD_EXECUTE;WORLD_READ;OWNER_EXECUTE;OWNER_READ;OWNER_WRITE;GROUP_EXECUTE;GROUP_READ;GROUP_WRITE
)
if(MSVC AND (BUILD_SHARED_LIBS))
	install(
		FILES $<TARGET_PDB_FILE:${PROJECT_NAME}>
		TYPE BIN
		OPTIONAL
	)
endif()

# Public Headers
install(
	DIRECTORY "${PROJECT_SOURCE_DIR}/include/."
	TYPE INCLUDE
)

# Install config.hpp
install(
	DIRECTORY "${PROJECT_BINARY_DIR}/generated/."
	TYPE INCLUDE
)

################################################################################
# Examples
################################################################################
if (${${PREFIX}BUILD_EXAMPLES})
	add_subdirectory("examples")
endif()

################################################################################
# Tests
################################################################################
if (${${PREFIX}BUILD_TESTS})
	add_subdirectory("tests")
endif()
