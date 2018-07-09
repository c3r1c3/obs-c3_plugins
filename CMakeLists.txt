cmake_minimum_required(VERSION 3.5)
project(obs-c3_plugins)

SET(CMAKE_INCLUDE_CURRENT_DIR ON)
SET(CMAKE_AUTOMOC ON)
SET(CMAKE_AUTOUIC ON)

SET(VERSION_MAJOR 0)
SET(VERSION_MINOR 1)
SET(VERSION_PATCH 0)
SET(VERSION_STR "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(ARCH 64)
	set(ARCH_NAME "64bit")
	set(OBS_BUILDDIR_ARCH "build64")
else()
	set(ARCH 32)
	set(ARCH_NAME "32bit")
	set(OBS_BUILDDIR_ARCH "build32")
endif()

# --- Set Complier Options ---
if(${CMAKE_C_COMPILER_ID} MATCHES "Clang" OR ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
	set(CMAKE_COMPILER_IS_CLANG TRUE)
endif()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_CLANG)
	set(CMAKE_CXX_FLAGS "-Wall -Wextra -Wno-unused-function -Werror-implicit-function-declaration -Wno-missing-field-initializers ${CMAKE_CXX_FLAGS} -fno-strict-aliasing")
	set(CMAKE_C_FLAGS "-Wall -Wextra -Wno-unused-function -Werror-implicit-function-declaration -Wno-missing-braces -Wno-missing-field-initializers ${CMAKE_C_FLAGS} -std=gnu99 -fno-strict-aliasing")

	option(USE_LIBC++ "Use libc++ instead of libstdc++" ${APPLE})
	if(USE_LIBC++)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
	endif()
elseif(MSVC)
	if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
		string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	else()
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
	endif()

	# Disable pointless constant condition warnings
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4127 /wd4201 /wd4456 /wd4457 /wd4458 /wd4459 /wd4595")
endif()

if(MSVC)
	set(CMAKE_C_FLAGS_DEBUG "/DDEBUG=1 /D_DEBUG=1 ${CMAKE_C_FLAGS_DEBUG}")
	set(CMAKE_CXX_FLAGS_DEBUG "/DDEBUG=1 /D_DEBUG=1 ${CMAKE_C_FLAGS_DEBUG}")

	if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SAFESEH:NO")
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /SAFESEH:NO")
		set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} /SAFESEH:NO")
	endif()
else()
	if(MINGW)
		set(CMAKE_WIDL "widl" CACHE STRING "wine IDL header file generation program")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_WIN32_WINNT=0x0600 -DWINVER=0x0600")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_WIN32_WINNT=0x0600 -DWINVER=0x0600")
	endif()
	set(CMAKE_C_FLAGS_DEBUG "-DDEBUG=1 -D_DEBUG=1 ${CMAKE_C_FLAGS_DEBUG}")
	set(CMAKE_CXX_FLAGS_DEBUG "-DDEBUG=1 -D_DEBUG=1 ${CMAKE_C_FLAGS_DEBUG}")
endif()
# --- End Complier options section ---

# --- Windows-specific build settings and tasks ---
if(MSVC AND NOT EXISTS "${CMAKE_BINARY_DIR}/ALL_BUILD.vcxproj.user")
	file(GENERATE
		OUTPUT "${CMAKE_BINARY_DIR}/ALL_BUILD.vcxproj.user"
		INPUT "${CMAKE_SOURCE_DIR}/cmake/ALL_BUILD.vcxproj.user.in")
endif()

if(WIN32)
	if (QTDIR OR DEFINED ENV{QTDIR} OR DEFINED ENV{QTDIR32} OR DEFINED ENV{QTDIR64})
		# Qt path set by user or env var
	else()
		set(QTDIR "" CACHE PATH "Path to Qt (e.g. C:/Qt/5.11/msvc2017_64)")
		message(WARNING "QTDIR variable is missing.  Please set this variable to specify path to Qt (e.g. C:/Qt/5.11/msvc2017_64)")
	endif()
	
	if (DepsPath OR DEFINED ENV{DepsPath} OR DEFINED ENV{DepsPath32} OR DEFINED ENV{DepsPath64})
		# Dependencies path set by user or env var
	else()
		set(DepsPath "" CACHE PATH "Path to compiled dependencies (e.g. D:/OBS-deps/win64)")
		message(WARNING "DepsPath variable is missing.  Please set this variable to specify path to compiled dependencies (e.g. D:/OBS-deps/win64)")
	endif()
	
	if(NOT DEFINED OBS_FRONTEND_LIB)
		set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
		message(FATAL_ERROR "Could not find OBS Frontend API\'s library !")
	endif()
	
	add_definitions(/O2)
	add_definitions(-DUNICODE -D_UNICODE -D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_WARNINGS)
	string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
	message("${CMAKE_CXX_FLAGS_RELEASE}")

	if(ARCH EQUAL 64)
		set(OBS_ARCH_NAME "64bit")
		set(OBS_BUILDDIR_ARCH "build64")
	else()
		set(OBS_ARCH_NAME "32bit")
		set(OBS_BUILDDIR_ARCH "build32")
		add_definitions(/arch:SSE2)
	endif()

	find_package(w32-pthreads REQUIRED)

	#Copy this section to the individual plug-in cmake files
	#add_custom_command(TARGET obs-c3_plugins POST_BUILD
	#	COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy
	#		"${CMAKE_BINARY_DIR}/$<CONFIG>")

	#	COMMAND if $<CONFIG:Debug>==1 ("${CMAKE_COMMAND}" -E copy
	#		"${CMAKE_BINARY_DIR}/$<CONFIG>")
	#)

	#Copy this section to the individual plug-in cmake files
	# --- Release package helper ---
	# The "release" folder has a structure similar OBS' one on Windows
	#set(RELEASE_DIR "${PROJECT_SOURCE_DIR}/release")

	#add_custom_command(TARGET obs-c3_plugins POST_BUILD
	#	COMMAND if $<CONFIG:Release>==1 (
	#		"${CMAKE_COMMAND}" -E make_directory
	#		"${RELEASE_DIR}/data/obs-plugins/obs-c3_plugins"
	#		"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")
			
	#	COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy_directory
	#		"${PROJECT_SOURCE_DIR}/data"
	#		"${RELEASE_DIR}/data/obs-plugins/obs-c3_plugins")

	#	COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy 
	#		"$<TARGET_FILE:obs-c3_plugins>"
	#		"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

	# Copy to obs-studio dev environment for immediate testing
	#	COMMAND if $<CONFIG:Debug>==1 (
	#		"${CMAKE_COMMAND}" -E copy
	#			"$<TARGET_FILE:obs-c3_plugins>"
	#			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}")

	#	COMMAND if $<CONFIG:Debug>==1 (
	#		"${CMAKE_COMMAND}" -E make_directory
	#			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-c3_plugins")

	#	COMMAND if $<CONFIG:Debug>==1 (
	#		"${CMAKE_COMMAND}" -E copy_directory
	#			"${PROJECT_SOURCE_DIR}/data"
	#			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-c3_plugins")
	#)
	# --- End of sub-section ---

endif()
# --- End of section ---

# --- Linux-specific build settings and tasks ---
if(UNIX AND NOT APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

	set_target_properties(obs-c3_plugins PROPERTIES PREFIX "")
	target_link_libraries(obs-c3_plugins
		obs-frontend-api)

	file(GLOB locale_files data/locale/*.ini)

	install(TARGETS obs-c3_plugins
		LIBRARY DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/obs-plugins")
	install(FILES ${locale_files}
		DESTINATION "${CMAKE_INSTALL_PREFIX}/share/obs/obs-plugins/obs-c3_plugins/locale")
endif()
# --- End of section ---

# --- OS X specific build settings and tasks ---
if(APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -fvisibility=default")
	set(CMAKE_MACOSX_RPATH TRUE)
	set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
	list(APPEND CMAKE_INSTALL_RPATH "@loader_path/" "@executable_path/")
elseif(UNIX)
	option(USE_XDG "Utilize XDG Base Directory Specification" ON)
	if(USE_XDG)
		add_definitions(-DUSE_XDG)
	endif()

	if(NOT UNIX_STRUCTURE)
		list(APPEND CMAKE_INSTALL_RPATH "$ORIGIN")
	endif()

	set_target_properties(obs-c3_plugins PROPERTIES PREFIX "")
	target_link_libraries(obs-c3_plugins "${OBS_FRONTEND_LIB}")
endif()
# --- End of section ---

# --- Platform-independent build settings ---
include(cmake/FindLibObs.cmake)
find_package(LibObs REQUIRED)
find_package(Qt5Core REQUIRED)
find_package(Qt5Widgets REQUIRED)
find_package(CXX11 REQUIRED)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX11_FLAGS}")

if(${CMAKE_C_COMPILER_ID} MATCHES "Clang" OR ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
	set(CMAKE_COMPILER_IS_CLANG TRUE)
endif()

configure_file(
	"${PROJECT_SOURCE_DIR}/headers/version.h.in"
	"${PROJECT_BINARY_DIR}/headers/version.h"
)

configure_file(
	"${PROJECT_SOURCE_DIR}/headers/c3-config.h.in"
	"${PROJECT_BINARY_DIR}/headers/c3-config.h"
)

file(COPY "${PROJECT_SOURCE_DIR}/headers/c3-plugin-defines.h"
     DESTINATION "${PROJECT_BINARY_DIR}/headers/"
)

include_directories(
	${LIBOBS_INCLUDE_DIR}
	"${LIBOBS_INCLUDE_DIR}/../UI/obs-frontend-api"
	${Qt5Core_INCLUDES}
	${Qt5Widgets_INCLUDES})
# --- End of section ---

# --- Start Subprojects Section ---
option(BUILD_AUDIO_FILTERS "Build Audio Filters" FALSE)
option(BUILD_DSK "Build Downstream Keyer" FALSE)
option(BUILD_FFMPEG_AUDIO "Build FFMPEG Audio Filters" FALSE)
option(BUILD_VISUAL_FILTERS "Build Visual Filters" FALSE)
option(BUILD_VST3 "Build VST3" FALSE)
option(BUILD_LV2 "Build LV2" FALSE)


if(BUILD_AUDIO_FILTERS)
	add_subdirectory(c3-audio-filters)
endif()

if(BUILD_DSK)
	add_subdirectory(c3-dsk)
endif()

if(BUILD_FFMPEG_AUDIO)
	add_subdirectory(c3-ffmpeg-audio-filters)
endif()

if(BUILD_VISUAL_FILTERS)
	add_subdirectory(c3-visual-filters)
endif()

if(BUILD_VST3)
	add_subdirectory(c3-vst3)
endif()

if (BUILD_LV2)
	add_subdirectory(c3-lv2)
endif()

set(ENABLE_PROGRAMS false)
# --- End Subprojects Section ---

# --- Start Project Generator Section ---
#if(MSVC AND NOT EXISTS "${CMAKE_BINARY_DIR}/ALL_BUILD.vcxproj.user")
#	file(GENERATE
#		OUTPUT "${CMAKE_BINARY_DIR}/ALL_BUILD.vcxproj.user"
#		INPUT "${CMAKE_SOURCE_DIR}/cmake/ALL_BUILD.vcxproj.user.in"
#		)
#endif()

if(NOT CMAKE_BUILD_TYPE)
        set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()
# --- End Project Generator Section ---
