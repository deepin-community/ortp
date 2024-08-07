############################################################################
# Copyright (c) 2010-2022 Belledonne Communications SARL.
#
# This file is part of oRTP 
# (see https://gitlab.linphone.org/BC/public/ortp).
#
############################################################################
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU Affero General Public License as
# published by the Free Software Foundation, either version 3 of the
# License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Affero General Public License for more details.
#
# You should have received a copy of the GNU Affero General Public License
# along with this program. If not, see <http://www.gnu.org/licenses/>.
############################################################################

cmake_minimum_required(VERSION 3.1)

# CMP0077 policy is required by Flexisip build. Remove it once the CMake required
# version is higer or equal to 3.13.
if(NOT CMAKE_VERSION VERSION_LESS 3.13)
    cmake_policy(SET CMP0077 NEW)
endif()

project(oRTP VERSION 5.2.0)



set(ORTP_MAJOR_VERSION ${PROJECT_VERSION_MAJOR})
set(ORTP_MINOR_VERSION ${PROJECT_VERSION_MINOR})
set(ORTP_MICRO_VERSION ${PROJECT_VERSION_PATCH})
set(ORTP_VERSION ${PROJECT_VERSION})
set(ORTP_SO_VERSION "15") # incremented for 4.4.0 version.
set(ORTP_DOC_VERSION "${ORTP_VERSION_MAJOR}.${ORTP_VERSION_MINOR}")


option(ENABLE_SHARED "Build shared library." YES)
option(ENABLE_STATIC "Build static library." YES)
option(ENABLE_DOC "Enable documentation generation with Doxygen." YES)
option(ENABLE_NTP_TIMESTAMP "Turn on NTP timestamping on packet reception." NO)
option(ENABLE_PERF "Disable costly features to reduce cpu consumtion and increase performance." NO)
option(ENABLE_STRICT "Build with strict compile options." YES)
option(ENABLE_TESTS "Enable compilation of test programs." NO)
option(ENABLE_UNIT_TESTS "Enable compilation of unit tests." YES)
option(ENABLE_DEBUG_LOGS "Turn on or off debug level logs." NO)
option(ENABLE_PACKAGE_SOURCE "Create 'package_source' target for source archive making (CMake >= 3.11)" OFF)
set(WITH_THREAD_STACK_SIZE "0" CACHE STRING "Set thread stack size (0 is the OS default).")


# Hidden non-cache options:
# * DISABLE_BC_PACKAGE_SEARCH: skip find_package() for every BC package (bctoolbox, ortp, etc.)


include(CheckIncludeFile)
include(CheckFunctionExists)
include(GNUInstallDirs)
include(CheckCSourceCompiles)
include(CheckCXXSourceCompiles)
include(CheckSymbolExists)

if(NOT CMAKE_INSTALL_RPATH AND CMAKE_INSTALL_PREFIX)
	set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_FULL_LIBDIR})
	message(STATUS "Setting install rpath to ${CMAKE_INSTALL_RPATH}")
endif()

set(MSVC_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include/MSVC")
if(MSVC)
	list(APPEND CMAKE_REQUIRED_INCLUDES "${MSVC_INCLUDE_DIR}")
endif()

find_package(Threads)
find_library(LIBM NAMES m)

if(NOT DISABLE_BC_PACKAGE_SEARCH)
	find_package(bctoolbox 0.2.0 REQUIRED)
endif()

check_include_file(sys/uio.h HAVE_SYS_UIO_H)
check_include_file(sys/audio.h HAVE_SYS_AUDIO_H)
if(NOT ANDROID)
	check_include_file(sys/shm.h HAVE_SYS_SHM_H)
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")# need this for check_cxx_source_compiles because CMAKE_CXX_STANDARD doesn't work.
check_cxx_source_compiles("#include <atomic>
using namespace std;
int main(int argc, char *argv[]) {
atomic_int current_ref;
atomic_init(&current_ref, 1);
atomic_int previous_ref(atomic_fetch_sub_explicit(&current_ref, 1, memory_order_release));
return 0;
}"
	HAVE_ATOMIC)
if(NOT HAVE_ATOMIC)
	message(FATAL_ERROR "Atomic(C++) libraries have not been found for ORTP.")
endif()

check_function_exists(arc4random HAVE_ARC4RANDOM)
check_symbol_exists(recvmsg "sys/socket.h" HAVE_RECVMSG)
check_symbol_exists(sendmsg "sys/socket.h" HAVE_SENDMSG)

include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)
if(WORDS_BIGENDIAN)
	set(ORTP_BIGENDIAN 1)
endif()


include_directories(
	include/
	src/
	${CMAKE_CURRENT_BINARY_DIR}
)
if(MSVC)
	include_directories(${MSVC_INCLUDE_DIR})
endif()


set(ORTP_CPPFLAGS ${BCTOOLBOX_CPPFLAGS})
if(ENABLE_STATIC)
	list(APPEND ORTP_CPPFLAGS "-DORTP_STATIC")
endif()
if(ENABLE_PERF)
	set(PERF 1)
endif()
if(ENABLE_NTP_TIMESTAMP)
	set(ORTP_TIMESTAMP 1)
	list(APPEND ORTP_CPPFLAGS "-DORTP_TIMESTAMP")
endif()
if(ENABLE_DEBUG_LOGS)
	set(ORTP_DEBUG_MODE 1)
endif()
if(CMAKE_USE_PTHREADS_INIT)
	set(ORTP_DEFAULT_THREAD_STACK_SIZE ${WITH_THREAD_STACK_SIZE})
endif()
if(APPLE)
	set(__APPLE_USE_RFC_3542 1)
endif()
if(ORTP_CPPFLAGS)
	add_definitions(${ORTP_CPPFLAGS})
endif()
set(POSIXTIMER_INTERVAL 10000)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/ortp-config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/ortp-config.h)
set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/ortp-config.h PROPERTIES GENERATED ON)
add_definitions(-DHAVE_CONFIG_H)

# Enable stdint.h limit macros on C++ files. (Windows only.)
if(MSVC)
	add_definitions("-D__STDC_LIMIT_MACROS")
endif()

set(STRICT_OPTIONS_CPP )
set(STRICT_OPTIONS_C )
if(MSVC)
	if(ENABLE_STRICT)
		list(APPEND STRICT_OPTIONS_CPP "/WX")
	endif()
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
else()
	list(APPEND STRICT_OPTIONS_CPP "-Wall" "-Wuninitialized")
	list(APPEND STRICT_OPTIONS_C "-Wdeclaration-after-statement" "-Wstrict-prototypes" "-Wno-error=strict-prototypes")
	if(CMAKE_C_COMPILER_ID MATCHES "Clang")
		list(APPEND STRICT_OPTIONS_CPP "-Qunused-arguments" "-Wno-array-bounds")
	endif()
	if(APPLE)
		list(APPEND STRICT_OPTIONS_CPP "-Wno-error=unknown-warning-option" "-Qunused-arguments" "-Wno-tautological-compare" "-Wno-unused-function" "-Wno-array-bounds")
	endif()
	if(ENABLE_STRICT)
		list(APPEND STRICT_OPTIONS_CPP "-Werror" "-Wextra" "-Wno-unused-parameter" "-Wno-missing-field-initializers" "-fno-strict-aliasing")
	endif()
endif()
if(STRICT_OPTIONS_CPP)
	list(REMOVE_DUPLICATES STRICT_OPTIONS_CPP)
endif()
if(STRICT_OPTIONS_C)
	list(REMOVE_DUPLICATES STRICT_OPTIONS_C)
endif()

set(EXPORT_TARGET_NAME ortp)

add_subdirectory(include)
add_subdirectory(src)
if(ENABLE_UNIT_TESTS)
	add_subdirectory(tester)
endif()

if(ENABLE_DOC)
	find_package(Doxygen)
	if(DOXYGEN_FOUND)
		set(srcdir ${CMAKE_CURRENT_SOURCE_DIR})
		configure_file(${CMAKE_CURRENT_SOURCE_DIR}/ortp.doxygen.in ${CMAKE_CURRENT_BINARY_DIR}/ortp.doxygen)
		file(GLOB DOC_INPUT_FILES
			include/ortp/[^.]*.h
			src/[^.]*.h
			src/[^.]*.c
		)
		add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/doc/html/index.html"
			COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/ortp.doxygen
			DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/ortp.doxygen ${DOC_INPUT_FILES}
		)
		add_custom_target(ortp-html-doc ALL DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/doc/html/index.html")
		install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/doc/html"
			DESTINATION "${CMAKE_INSTALL_DATADIR}/doc/ortp-${ORTP_VERSION}")
	endif()
endif()




set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix ${prefix}/bin)
set(libdir ${prefix}/lib)
set(includedir ${prefix}/include)
set(ORTP_PKGCONFIG_VERSION "${ORTP_VERSION}")
set(ORTPDEPS_LIBS )

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/ortp.pc.in ${CMAKE_CURRENT_BINARY_DIR}/ortp.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/ortp.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

if (ENABLE_PACKAGE_SOURCE)
	add_subdirectory(build)
endif()

include(CMakePackageConfigHelpers)

set(CONFIG_PACKAGE_LOCATION "${CMAKE_INSTALL_LIBDIR}/cmake/${EXPORT_TARGET_NAME}")

write_basic_package_version_file(
	"${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_TARGET_NAME}ConfigVersion.cmake"
	VERSION ${ORTP_VERSION}
	COMPATIBILITY AnyNewerVersion
)
if(LINPHONE_BUILDER_GROUP_EXTERNAL_SOURCE_PATH_BUILDERS)
	export(EXPORT "${EXPORT_TARGET_NAME}Targets"
		FILE "${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_TARGET_NAME}Targets.cmake"
	)
endif()

configure_package_config_file(ORTPConfig.cmake.in
	"${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_TARGET_NAME}Config.cmake"
	INSTALL_DESTINATION ${CONFIG_PACKAGE_LOCATION}
	NO_SET_AND_CHECK_MACRO
)

install(EXPORT ${EXPORT_TARGET_NAME}Targets
	FILE "${EXPORT_TARGET_NAME}Targets.cmake"
	DESTINATION ${CONFIG_PACKAGE_LOCATION}
)

install(FILES
	"${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_TARGET_NAME}Config.cmake"
	"${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_TARGET_NAME}ConfigVersion.cmake"
	DESTINATION ${CONFIG_PACKAGE_LOCATION}
)
install(FILES 	"${CMAKE_CURRENT_SOURCE_DIR}/README.md" 
	"${CMAKE_CURRENT_SOURCE_DIR}/CHANGELOG.md" 
	"${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt" 
	"${CMAKE_CURRENT_SOURCE_DIR}/AUTHORS.md" 
	DESTINATION "${CMAKE_INSTALL_DATADIR}/doc/ortp-${ORTP_VERSION}"
)
