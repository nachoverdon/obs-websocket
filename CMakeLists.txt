cmake_minimum_required(VERSION 3.2)
project(obs-websocket)

set(CMAKE_PREFIX_PATH "${QTDIR}")
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if (WIN32 OR APPLE)
include(external/FindLibObs.cmake)
endif()

add_definitions(-DASIO_STANDALONE)

if (UNIX)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

if (WIN32 OR APPLE)
	include(external/FindLibObs.cmake)
endif()

find_package(LibObs REQUIRED)
find_package(Qt5Core REQUIRED)
find_package(Qt5Widgets REQUIRED)
find_package(Boost REQUIRED)

set(obs-websocket_SOURCES
	src/obs-websocket.cpp
	src/WSServer.cpp
	src/WSRequestHandler.cpp
	src/WSRequestHandler_General.cpp
	src/WSRequestHandler_Profiles.cpp
	src/WSRequestHandler_Recording.cpp
	src/WSRequestHandler_ReplayBuffer.cpp
	src/WSRequestHandler_SceneCollections.cpp
	src/WSRequestHandler_Scenes.cpp
	src/WSRequestHandler_SceneItems.cpp
	src/WSRequestHandler_Sources.cpp
	src/WSRequestHandler_Streaming.cpp
	src/WSRequestHandler_StudioMode.cpp
	src/WSRequestHandler_Transitions.cpp
	src/WSEvents.cpp
	src/Config.cpp
	src/Utils.cpp
	src/forms/settings-dialog.cpp)

set(obs-websocket_HEADERS
	src/obs-websocket.h
	src/WSServer.h
	src/WSRequestHandler.h
	src/WSEvents.h
	src/Config.h
	src/Utils.h
	src/forms/settings-dialog.h)

# --- Platform-independent build settings ---
add_library(obs-websocket MODULE
	${obs-websocket_SOURCES}
	${obs-websocket_HEADERS})

include_directories(
	"${LIBOBS_INCLUDE_DIR}/../UI/obs-frontend-api"
	${Qt5Core_INCLUDES}
	${Qt5Widgets_INCLUDES}
	${Boost_INCLUDE_DIRS}
	"${CMAKE_SOURCE_DIR}/deps/asio/asio/include"
	"${CMAKE_SOURCE_DIR}/deps/websocketpp")

target_link_libraries(obs-websocket
	libobs
	Qt5::Core
	Qt5::Widgets)

# --- End of section ---

# --- Windows-specific build settings and tasks ---
if(WIN32)
	if(NOT DEFINED OBS_FRONTEND_LIB)
		set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
		message(FATAL_ERROR "Could not find OBS Frontend API's library !")
	endif()

	add_definitions(-D_WEBSOCKETPP_CPP11_STL_)

	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(ARCH_NAME "64bit")
		set(OBS_BUILDDIR_ARCH "build64")
	else()
		set(ARCH_NAME "32bit")
		set(OBS_BUILDDIR_ARCH "build32")
	endif()

	include_directories(
		"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/UI"
	)

	target_link_libraries(obs-websocket
		"${OBS_FRONTEND_LIB}")

	# --- Release package helper ---
	# The "release" folder has a structure similar OBS' one on Windows
	set(RELEASE_DIR "${PROJECT_SOURCE_DIR}/release")

	add_custom_command(TARGET obs-websocket POST_BUILD
		# If config is Release, package release files
		COMMAND if $<CONFIG:Release>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${RELEASE_DIR}/data/obs-plugins/obs-websocket"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

		COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy_directory
			"${PROJECT_SOURCE_DIR}/data"
			"${RELEASE_DIR}/data/obs-plugins/obs-websocket")

		COMMAND if $<CONFIG:Release>==1 ("${CMAKE_COMMAND}" -E copy
			"$<TARGET_FILE:obs-websocket>"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

		# If config is RelWithDebInfo, package release files
		COMMAND if $<CONFIG:RelWithDebInfo>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${RELEASE_DIR}/data/obs-plugins/obs-websocket"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

		COMMAND if $<CONFIG:RelWithDebInfo>==1 ("${CMAKE_COMMAND}" -E copy_directory
			"${PROJECT_SOURCE_DIR}/data"
			"${RELEASE_DIR}/data/obs-plugins/obs-websocket")

		COMMAND if $<CONFIG:RelWithDebInfo>==1 ("${CMAKE_COMMAND}" -E copy
			"$<TARGET_FILE:obs-websocket>"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

		COMMAND if $<CONFIG:RelWithDebInfo>==1 ("${CMAKE_COMMAND}" -E copy
			"$<TARGET_PDB_FILE:obs-websocket>"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}")

		# Copy to obs-studio dev environment for immediate testing
		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy
				"$<TARGET_FILE:obs-websocket>"
				"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}")

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy
				"$<TARGET_PDB_FILE:obs-websocket>"
				"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}")

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E make_directory
				"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-websocket")

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy_directory
				"${PROJECT_SOURCE_DIR}/data"
				"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-websocket")
	)
	# --- End of sub-section ---

endif()
# --- End of section ---

# --- Linux-specific build settings and tasks ---
if(UNIX AND NOT APPLE)
	set_target_properties(obs-websocket PROPERTIES PREFIX "")
	target_link_libraries(obs-websocket obs-frontend-api)

	file(GLOB locale_files data/locale/*.ini)
	execute_process(COMMAND uname -m COMMAND tr -d '\n' OUTPUT_VARIABLE UNAME_MACHINE)

	install(TARGETS obs-websocket
		LIBRARY DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/obs-plugins")
	# Dirty fix for Ubuntu
        install(TARGETS obs-websocket
                LIBRARY DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/${UNAME_MACHINE}-linux-gnu/obs-plugins")

	install(FILES ${locale_files}
		DESTINATION "${CMAKE_INSTALL_PREFIX}/share/obs/obs-plugins/obs-websocket/locale")
endif()
# --- End of section ---

# -- OS X specific build settings and tasks --
if(APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -fvisibility=default")

	set(CMAKE_SKIP_RPATH TRUE)
	set_target_properties(obs-websocket PROPERTIES PREFIX "")
	target_link_libraries(obs-websocket "${OBS_FRONTEND_LIB}")
endif()
# -- End of section --
