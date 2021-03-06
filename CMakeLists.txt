cmake_minimum_required(VERSION 3.5)
project(tuna VERSION 1.5.0)

add_subdirectory(deps)

set(PLUGIN_AUTHOR "univrsal")
set(PLUGIN_GIT tuna)
set(LINUX_MAINTAINER_EMAIL "uni@vrsal.cf")
set(MACOS_BUNDLEID "cf.vrsal.tuna")
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_PREFIX_PATH "${QTDIR}")
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

string(TIMESTAMP TODAY "%Y.%m.%d %H:%M")
add_definitions(-DBUILD_TIME="${TODAY}")
add_definitions(-DTUNA_VERSION="${PROJECT_VERSION}")

include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/ObsPluginHelpers.cmake)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/external")
if (MSVC OR APPLE)
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/obs")
endif()

find_package(LibObs REQUIRED)
find_package(Qt5Widgets REQUIRED)
find_package(Libcurl REQUIRED)
find_package(LibVLC REQUIRED)

option(LOCAL_INSTALLATION "Whether to install for the current user (default: OFF)" OFF)
option(GLOBAL_INSTALLATION "Whether to install for all users (default: OFF)" OFF)

include_directories(${LIBCURL_INCLUDE_DIRS})
include_directories(${TAGLIB_INCLUDE_DIRS})
include_directories(${LIBMPDCLIENT_INCLUDE_DIRS})

if (OBS_FRONTEND_INCLUDE)
    include_directories(${OBS_FRONTEND_INCLUDE})
endif()

add_definitions(${LIBCURL_DEFINITIONS})

configure_file(
    package/installer-macOS.pkgproj.in
    ../package/installer-macOS.pkgproj
)
configure_file(
    package/installer-Windows.iss.in
    ../package/installer-Windows.iss
)
configure_file(
    package/README.txt.in
    ../package/README.txt
)

if (NOT CREDS)
    set(CREDS "MISSING")
    message(WARNING "Missing Spotify API info")
endif()

if (NOT LASTFM_CREDS)
    set(LASTFM_CREDS "MISSING")
    message(WARNING "Missing Last.fm API info")
endif()

mark_as_advanced(LASTFM_CREDS)
mark_as_advanced(CREDS)

add_definitions(-DSPOTIFY_CREDENTIALS=\"${CREDS}\")
add_definitions(-DLASTFM_CREDENTIALS=\"${LASTFM_CREDS}\")

set(tuna_ui
        src/gui/tuna_gui.ui
        src/gui/output_edit_dialog.ui
        src/gui/music_control.ui
        src/gui/widgets/lastfm.ui
        src/gui/widgets/mpd.ui
        src/gui/widgets/window_title.ui
        src/gui/widgets/spotify.ui
        src/gui/widgets/vlc.ui)

qt5_wrap_ui(tuna_ui_headers ${tuna_ui})
qt5_add_resources(tuna_qrc_sources src/gui/tuna.qrc)

if (UNIX AND NOT APPLE)
    add_definitions(-DLINUX=1)
    set(tuna_platform_sources "./src/util/window/window_helper_nix.cpp")
    set(LIBOBS_LIBRARIES "libobs")
    set(OBS_FRONTEND_LIB "obs-frontend-api")
endif()

if (APPLE)
    add_definitions(-DMACOS=1)
    set(tuna_platform_sources "./src/util/window/window_helper_mac.mm")
    find_library(COCOA Cocoa)

    include_directories(${COCOA})
    set_source_files_properties("./src/util/window/window_helper_mac.mm"
        PROPERTIES COMPILE_FLAGS "-fobjc-arc")

    set(tuna_platform_deps
        ${COCOA})
    set(FRONTEND_LIB "${OBS_FRONTEND_LIB}")
else()
    set(FRONTEND_LIB "obs-frontend-api")
    set(OBS_LIB "libobs")
endif()

if (UNIX)
    add_definitions(-DUNIX=1)
endif ()

if (MSVC)
    set(tuna_platform_sources "./src/util/window/window_helper_win.cpp")
    set(tuna_platform_deps
        ${W32_PTHREADS_LIB}
        wsock32)
endif ()

if (NOT LibVLC_FOUND AND ENABLE_VLC)
    message(STATUS "LibVLC includes not found but set as enabled")
    add_definitions(-DDISABLE_TUNA_VLC=1)
    set(DISABLEVLC TRUE)
elseif (NOT LibVLC_FOUND)
    message(STATUS "LibVLC includes not found, VLC source support disabled")
    message(STATUS "If you want VLC support, clone the VLC repo and point VLC_INCLUDE_DIR to <repo>/include/vlc ")
    set(DISABLEVLC TRUE)
    add_definitions(-DDISABLE_TUNA_VLC=1)
else()
    message(STATUS "LibVLC headers found, VLC source support enabled")
endif()

if (DISABLE_VLC)
    message(STATUS "VLC integration disabled")
    add_definitions(-DDISABLE_TUNA_VLC=1)
else()
    include_directories(${LIBVLC_INCLUDE_DIRS})
    message(STATUS "Using VLC headers: ${LIBVLC_INCLUDE_DIRS}")
    add_definitions(${LIBVLC_DEFINITIONS})
    set(tuna_vlc_source
        ./src/query/vlc_obs_source.cpp
        ./src/util/vlc_internal.h
        ./src/util/vlc_internal.c)
endif()

set(tuna_sources
    ./src/tuna_plugin.cpp
    ./src/util/constants.hpp
    ./src/util/config.cpp
    ./src/util/config.hpp
    ./src/util/creds.hpp
    ./src/gui/tuna_gui.cpp
    ./src/gui/tuna_gui.hpp
    ./src/gui/output_edit_dialog.cpp
    ./src/gui/output_edit_dialog.hpp
    ./src/gui/music_control.cpp
    ./src/gui/music_control.hpp
    ./src/gui/scrolltext.cpp
    ./src/gui/scrolltext.hpp
    ./src/query/music_source.hpp
    ./src/query/music_source.cpp
    ./src/query/spotify_source.cpp
    ./src/query/spotify_source.hpp
    ./src/query/mpd_source.cpp
    ./src/query/mpd_source.hpp
    ./src/query/window_source.cpp
    ./src/query/window_source.hpp
    ./src/query/lastfm_source.cpp
    ./src/query/lastfm_source.hpp
    ./src/query/gpmdp_source.cpp
    ./src/query/gpmdp_source.hpp
    ./src/query/song.cpp
    ./src/query/song.hpp
    ./src/util/format.cpp
    ./src/util/format.hpp
    ./src/source/progress.cpp
    ./src/source/progress.hpp
    ./src/util/cover_tag_handler.cpp
    ./src/util/cover_tag_handler.hpp
    ./src/query/vlc_obs_source.hpp
    ./src/util/tuna_thread.cpp
    ./src/util/tuna_thread.hpp
    ./src/util/utility.cpp
    ./src/util/utility.hpp
    ./src/util/window/window_helper.hpp
    ./src/gui/widgets/lastfm.cpp
    ./src/gui/widgets/lastfm.hpp
    ./src/gui/widgets/mpd.cpp
    ./src/gui/widgets/mpd.hpp
    ./src/gui/widgets/window_title.cpp
    ./src/gui/widgets/window_title.hpp
    ./src/gui/widgets/spotify.cpp
    ./src/gui/widgets/spotify.hpp
    ./src/gui/widgets/vlc.cpp
    ./src/gui/widgets/vlc.hpp
    ${tuna_ui_headers}
    ${tuna_vlc_source}
    )

add_library(tuna MODULE
    ${tuna_sources}
    ${tuna_ui}
    ${tuna_platform_sources}
    ${tuna_qrc_sources})

include_directories(
    "${LIBOBS_INCLUDE_DIR}/../UI/obs-frontend-api"
    ${LIBOBS_INCLUDE_DIR}
    ${Qt5Core_INCLUDES}
    ${Qt5Widgets_INCLUDES}
)

target_link_libraries(tuna
    ${LIBOBS_LIBRARIES}
    tag
    mpdclient
    Qt5::Widgets
    Qt5::Core
    ${OBS_FRONTEND_LIB}
    ${LIBCURL_LIBRARIES}
    ${tuna_platform_deps})

# --- Windows-specific build settings and tasks ---
if(WIN32)
    if(NOT DEFINED OBS_FRONTEND_LIB)
        set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
        message(FATAL_ERROR "Could not find OBS Frontend API\'s library !")
    endif()

    # Enable Multicore Builds and disable FH4 (to not depend on VCRUNTIME140_1.DLL when building with VS2019)
    if (MSVC)
        add_definitions(/MP /d2FH4-)
    endif()

    include_directories(
        "${LIBOBS_INCLUDE_DIR}"
    )

    target_link_libraries(${CMAKE_PROJECT_NAME}
        "${OBS_FRONTEND_LIB}"
    )

    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(ARCH_NAME "64bit")
        set(OBS_BUILDDIR_ARCH "build64")
    else()
        set(ARCH_NAME "32bit")
        set(OBS_BUILDDIR_ARCH "build32")
    endif()

    get_filename_component(LibOBS_DIR ${LIBOBS_LIB} DIRECTORY)

    add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
        # Copy to obs-studio dev environment for immediate testing
        COMMAND if $<CONFIG:Debug>==1 (
            "${CMAKE_COMMAND}" -E copy
                "$<TARGET_FILE:${CMAKE_PROJECT_NAME}>"
                "${LibOBS_DIR}/../../rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}")

        COMMAND if $<CONFIG:Debug>==1 (
            "${CMAKE_COMMAND}" -E copy
                "$<TARGET_PDB_FILE:${CMAKE_PROJECT_NAME}>"
                "${LibOBS_DIR}/../../rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}")

        COMMAND if $<CONFIG:Debug>==1 (
            "${CMAKE_COMMAND}" -E make_directory
                "${LibOBS_DIR}/../../rundir/$<CONFIG>/data/obs-plugins/${CMAKE_PROJECT_NAME}")

        COMMAND if $<CONFIG:Debug>==1 (
            "${CMAKE_COMMAND}" -E copy_directory
                "${PROJECT_SOURCE_DIR}/data"
                "${LibOBS_DIR}/../../rundir/$<CONFIG>/data/obs-plugins/${CMAKE_PROJECT_NAME}")
    )
endif()

if(UNIX AND NOT APPLE)
    include(GNUInstallDirs)

    set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES PREFIX "")
    target_link_libraries(${CMAKE_PROJECT_NAME} obs-frontend-api)
    if (NOT OBS_ARCH_NAME)
        message(FATAL_ERROR "Can't install plugin without knowing architecture")
    endif()

    if (LOCAL_INSTALLATION)
        # Installs into home directory
        install(TARGETS ${CMAKE_PROJECT_NAME}
            LIBRARY DESTINATION "$ENV{HOME}/.config/obs-studio/plugins/${CMAKE_PROJECT_NAME}/bin/${OBS_ARCH_NAME}")

        install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/data
            DESTINATION "$ENV{HOME}/.config/obs-studio/plugins/${CMAKE_PROJECT_NAME}")
    elseif(GLOBAL_INSTALLATION)
        # For *.deb installer
        install(TARGETS ${CMAKE_PROJECT_NAME}
            LIBRARY DESTINATION "/usr/lib/obs-plugins") # hardcoded, since ubuntu uses the wrong path by default

        file(GLOB locale_files data/locale/*.ini)
        file(GLOB other_files data/*.*)

        install(FILES ${locale_files}
            DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/obs/obs-plugins/${CMAKE_PROJECT_NAME}/locale")
        install(FILES ${other_files}
            DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/obs/obs-plugins/${CMAKE_PROJECT_NAME}")
    else()
        # For *.zip binary
        install(TARGETS ${CMAKE_PROJECT_NAME}
            LIBRARY DESTINATION "${CMAKE_PROJECT_NAME}/bin/${OBS_ARCH_NAME}")

        install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/data
            DESTINATION "${CMAKE_PROJECT_NAME}")

        install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/package/README.txt"
            DESTINATION "${CMAKE_PROJECT_NAME}")
    endif()
elseif(APPLE)
    if(NOT DEFINED OBS_FRONTEND_LIB)
        set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
        message(FATAL_ERROR "Could not find OBS Frontend API\'s library !")
    endif()

    target_link_libraries(${CMAKE_PROJECT_NAME}
        "${OBS_FRONTEND_LIB}"
    )
endif()
