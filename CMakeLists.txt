cmake_minimum_required (VERSION 3.0)
project (edb CXX)
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND "${CMAKE_CXX_COMPILER_VERSION}" VERSION_LESS 5.0)
	message(FATAL_ERROR "Your g++ version is too old. At least 5.0 is required.")
endif()

if(NOT EXISTS "${PROJECT_SOURCE_DIR}/src/qhexview/")
	message(SEND_ERROR "The git submodules are not available. Please run:\ngit submodule update --init --recursive"
)
endif()

option(ENABLE_ASAN      "Enable address santiziers")
option(ENABLE_USAN      "Enable undefined santiziers")
option(ENABLE_MSAN      "Enable memory santiziers")
option(ENABLE_TSAN      "Enable thread santiziers")
option(ENABLE_STL_DEBUG "Enable STL container debugging")

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake/Modules/")

include("GNUInstallDirs")
include("CheckIncludeFileCXX")

find_package(Boost 1.35 REQUIRED)
include_directories(${Boost_INCLUDE_DIR})

find_package(Capstone REQUIRED)
include_directories(${CAPSTONE_INCLUDE_DIRS})
link_directories(${CAPSTONE_LIBRARY_DIRS})

# eventually use this to generate version.h?
set(TARGET_VERSION_MAJOR 1)
set(TARGET_VERSION_MINOR 0)
set(TARGET_VERSION_PATCH 0)
set_property(GLOBAL PROPERTY VERSION ${TARGET_VERSION_MAJOR}.${TARGET_VERSION_MINOR}.${TARGET_VERSION_PATCH})


if (UNIX)
	find_package(PkgConfig REQUIRED)
	pkg_check_modules(GRAPHVIZ libgvc>=2.38.0)
	if(${GRAPHVIZ_FOUND})
		include_directories(${GRAPHVIZ_INCLUDE_DIRS})
		link_directories(${GRAPHVIZ_LIBRARY_DIRS})
		add_definitions(-DENABLE_GRAPH)
	endif()
endif()

message(STATUS "Checking for module 'double-conversion'")
# Using check_include_file_cxx instead of find_package etc. to support various versions of double-conversion. Some versions don't have CMake modules, some have a bit incompatible ones...
check_include_file_cxx("double-conversion/double-conversion.h" HAVE_DOUBLE_CONVERSION)
if(NOT HAVE_DOUBLE_CONVERSION)
   UNSET(HAVE_DOUBLE_CONVERSION CACHE)
   message(WARNING      "libdouble-conversion header wasn't found. 32- and 64-bit floating-point values will be showed with max_digits10 digits of precision instead of shortest representation.")
else()
   find_library(DOUBLE_CONVERSION_LIBRARIES double-conversion)
   if(NOT DOUBLE_CONVERSION_LIBRARIES)
       message(WARNING "libdouble-conversion library wasn't found. 32- and 64-bit floating-point values will be showed with max_digits10 digits of precision instead of shortest representation.")
   else()
       add_definitions("-DHAVE_DOUBLE_CONVERSION")
   endif()
endif()

find_package(Qt5Core)

include_directories("include")

if(UNIX)
	include_directories("include/os/unix")
elseif(WIN32)
	include_directories("include/os/win32")
endif()

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	include_directories("include/os/unix/linux")
endif()

if(${CMAKE_SIZEOF_VOID_P} EQUAL 4)
	add_definitions(-DEDB_IS_32_BIT=true -DEDB_IS_64_BIT=false)
elseif(${CMAKE_SIZEOF_VOID_P} EQUAL 8)
	add_definitions(-DEDB_IS_32_BIT=false -DEDB_IS_64_BIT=true)
else()
	message(SEND_ERROR "Unexpected bitness: \"sizeof(void*)=${CMAKE_SIZEOF_VOID_P}.\"")
endif()

if((${CMAKE_SYSTEM_PROCESSOR} MATCHES "i[3456]86") OR (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64") OR (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "AMD64"))
	set(CPU_TYPE "x86")
	if(${CMAKE_SIZEOF_VOID_P} EQUAL 4)
		add_definitions(-DEDB_X86)
	elseif(${CMAKE_SIZEOF_VOID_P} EQUAL 8)
		add_definitions(-DEDB_X86_64)
	endif()
	include_directories("include/arch/x86-generic")
endif()

if((${CMAKE_SYSTEM_PROCESSOR} MATCHES "armv[0-9]+"))
	set(CPU_TYPE "arm")
	if(${CMAKE_SIZEOF_VOID_P} EQUAL 4)
		add_definitions(-DEDB_ARM32)
	elseif(${CMAKE_SIZEOF_VOID_P} EQUAL 8)
		add_definitions(-DEDB_ARM64)
	endif()
	include_directories("include/arch/arm-generic")
endif()

# FIXME: This is also useful on Windows, so it should be made usable there too.
if(UNIX)
	if(${CPU_TYPE} STREQUAL "x86")
		pkg_check_modules(GDTOA gdtoa-desktop)
		if(NOT GDTOA_FOUND)
			message(WARNING "gdtoa-desktop package wasn't found. 80-bit floating-point values will be showed with max_digits10 digits of precision instead of shortest representation.")
		else()
			add_definitions("-DHAVE_GDTOA")
			include_directories(${GDTOA_INCLUDE_DIRS})
			link_directories(${GDTOA_LIBRARY_DIRS})
		endif()
	endif()
endif()


if ((${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang") OR (${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU"))
	if(ENABLE_ASAN)
		set(CMAKE_CXX_FLAGS        "${CMAKE_CXX_FLAGS}        -fsanitize=address") # -fsanitize-address-use-after-scope
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
	endif()

	if(ENABLE_USAN)
		set(CMAKE_CXX_FLAGS        "${CMAKE_CXX_FLAGS}        -fsanitize=undefined")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=undefined")
	endif()

	if(ENABLE_TSAN)
		set(CMAKE_CXX_FLAGS        "${CMAKE_CXX_FLAGS}        -fsanitize=thread")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=thread")
	endif()

	if(ENABLE_MSAN)
		set(CMAKE_CXX_FLAGS        "${CMAKE_CXX_FLAGS}        -fsanitize=memory")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=memory")
	endif()
	
	if(ENABLE_STL_DEBUG)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_DEBUG")
		set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -D_GLIBCXX_DEBUG")	
	endif()
	
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -W -Wall -pedantic -Wunused")
	set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -W -Wall -pedantic -Wunused")
endif()

if(MSVC)
  add_definitions(-DUNICODE -D_UNICODE)
endif()

add_subdirectory(src)
add_subdirectory(plugins)

install (FILES ${CMAKE_SOURCE_DIR}/edb.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
install (FILES ${CMAKE_SOURCE_DIR}/edb.desktop DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications/)
install (FILES ${CMAKE_SOURCE_DIR}/src/images/edb.png DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/pixmaps/)
