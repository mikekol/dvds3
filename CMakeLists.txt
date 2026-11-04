cmake_minimum_required(VERSION 3.28)

# Version comes from version.json: "version" is the MAJOR.MINOR.PATCH release (edited by hand),
# "build" is incremented by the .githooks/pre-commit hook on every commit.
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/version.json" _version_json)
string(JSON _base_version GET "${_version_json}" version)
string(JSON _build_number GET "${_version_json}" build)
set(PLUGIN_VERSION_FULL "${_base_version}+${_build_number}")

project(dvd-screensaver VERSION ${_base_version} LANGUAGES C)

set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/version.json")

# Register the git hooks (the CMake equivalent of an npm "prepare" script)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
    execute_process(
        COMMAND git config core.hooksPath .githooks
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        OUTPUT_QUIET ERROR_QUIET)
endif()

set(PLUGIN_AUTHOR "univrsal")
set(PLUGIN_GIT dvds3)
set(LINUX_MAINTAINER_EMAIL "uni@vrsal.xyz")
set(MACOS_BUNDLEID "xyz.vrsal.dvds3")

option(LOCAL_INSTALLATION "Whether to install for the current user (default: OFF)" OFF)
option(GLOBAL_INSTALLATION "Whether to install for all users (default: OFF)" OFF)

if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(FATAL_ERROR "32-bit builds are not supported; configure for a 64-bit target (e.g. -A x64)")
endif()
set(OBS_ARCH_NAME "64bit")

configure_file(
    package/installer-macOS.pkgproj.in
    ${CMAKE_SOURCE_DIR}/package/installer-macOS.pkgproj
)
configure_file(
    package/installer-Windows.wxs.in
    ${CMAKE_SOURCE_DIR}/package/installer-Windows.wxs
)

# WiX shows its license page as RTF: wrap COPYING.txt in a minimal RTF document
file(READ "${CMAKE_SOURCE_DIR}/COPYING.txt" _license_text)
string(REPLACE "\\" "\\\\" _license_text "${_license_text}")
string(REPLACE "{" "\\{" _license_text "${_license_text}")
string(REPLACE "}" "\\}" _license_text "${_license_text}")
string(REPLACE "\r" "" _license_text "${_license_text}")
string(REPLACE "\n" "\\par\n" _license_text "${_license_text}")
file(WRITE "${CMAKE_SOURCE_DIR}/package/LICENSE.rtf"
    "{\\rtf1\\ansi\\deff0{\\fonttbl{\\f0 Consolas;}}\\f0\\fs18\n${_license_text}}\n")
configure_file(
    package/README.txt.in
    ${CMAKE_SOURCE_DIR}/package/README.txt
)

# The only dependency: libobs from OBS Studio 33+, via its exported CMake package.
# Point libobs_DIR at <obs-install-prefix>/lib/cmake/libobs (or the build tree equivalent).
find_package(libobs 33 REQUIRED)

configure_file(src/version.h.in ${CMAKE_CURRENT_BINARY_DIR}/generated/version.h @ONLY)

add_library(dvd-screensaver MODULE src/dvd.c)
target_include_directories(dvd-screensaver PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/generated)
target_link_libraries(dvd-screensaver PRIVATE OBS::libobs)

if(UNIX AND NOT APPLE)
    target_compile_definitions(dvd-screensaver PRIVATE UNIX=1)
    set_target_properties(dvd-screensaver PROPERTIES PREFIX "")
endif()

if(APPLE)
    target_compile_definitions(dvd-screensaver PRIVATE MAC_OS=1)
endif()

if(MSVC)
    # Enable multicore builds and disable FH4 (no VCRUNTIME140_1.DLL dependency)
    target_compile_options(dvd-screensaver PRIVATE /MP /d2FH4-)
endif()

include(GNUInstallDirs)

if(WIN32)
    install(TARGETS dvd-screensaver
        RUNTIME DESTINATION "obs-plugins/${OBS_ARCH_NAME}"
        LIBRARY DESTINATION "obs-plugins/${OBS_ARCH_NAME}")
    install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/data/
        DESTINATION "data/obs-plugins/${CMAKE_PROJECT_NAME}")
elseif(UNIX AND NOT APPLE)
    if(LOCAL_INSTALLATION)
        # Installs into home directory
        install(TARGETS dvd-screensaver
            LIBRARY DESTINATION "$ENV{HOME}/.config/obs-studio/plugins/${CMAKE_PROJECT_NAME}/bin/${OBS_ARCH_NAME}")
        install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/data
            DESTINATION "$ENV{HOME}/.config/obs-studio/plugins/${CMAKE_PROJECT_NAME}/")
    elseif(GLOBAL_INSTALLATION)
        # For *.deb installer
        install(TARGETS dvd-screensaver
            LIBRARY DESTINATION "/usr/lib/obs-plugins") # Ubuntu apparently doesn't know the correct path

        file(GLOB locale_files data/locale/*.ini)
        file(GLOB other_files data/*.*)

        install(FILES ${locale_files}
            DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/obs/obs-plugins/${CMAKE_PROJECT_NAME}/locale")
        install(FILES ${other_files}
            DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/obs/obs-plugins/${CMAKE_PROJECT_NAME}")
    else()
        # For *.zip binary
        install(TARGETS dvd-screensaver
            LIBRARY DESTINATION "${CMAKE_PROJECT_NAME}/bin/${OBS_ARCH_NAME}")
        install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/data
            DESTINATION "${CMAKE_PROJECT_NAME}")
        install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/package/README.txt"
            DESTINATION "${CMAKE_PROJECT_NAME}")
    endif()
endif()
