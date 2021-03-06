################################################################################
#
#  Copyright (c) 2010-2020 Belledonne Communications SARL.
# 
#  This file is part of linphone-desktop
#  (see https://www.linphone.org).
# 
#  This program is free software: you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation, either version 3 of the License, or
#  (at your option) any later version.
# 
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
# 
#  You should have received a copy of the GNU General Public License
#  along with this program. If not, see <http://www.gnu.org/licenses/>.
#
################################################################################

# First building command:
#   cmake .. && cmake --build . --target all --parallel 10 && cmake --build . --target install && cmake ..
# Logic : Prepare first building of the SDK and Minizip. Install binaries and prepare for application building.
# Next, we can build the application with cmake --build . --target all --parallel 10 && cmake --build . --target install

cmake_minimum_required(VERSION 3.1)
project(linphoneqt VERSION 4.1.1)

# Prepare gobal CMAKE configuration specific to the current project
set(SDK_BUILD_DIR "${CMAKE_BINARY_DIR}/WORK")       # SDK build in WORK. Keep all in it.
set(LINPHONE_OUTPUT_DIR "${CMAKE_BINARY_DIR}/linphone-sdk/desktop") #This is fixed by the SDK
set(MINIZIP_OUTPUT_DIR "${CMAKE_BINARY_DIR}/minizip_OUTPUT")

set(APPLICATION_OUTPUT_DIR "${CMAKE_BINARY_DIR}/OUTPUT")

message("minizip=${MINIZIP_OUTPUT_DIR} sdk=${LINPHONE_OUTPUT_DIR}")

list(APPEND CMAKE_PREFIX_PATH "${LINPHONE_OUTPUT_DIR};${MINIZIP_OUTPUT_DIR};${APPLICATION_OUTPUT_DIR}")

if(UNIX AND NOT APPLE)
    set(CMAKE_INSTALL_RPATH "$ORIGIN;$ORIGIN/lib64;$ORIGIN/../lib64;$ORIGIN/lib;$ORIGIN/../lib")
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
endif()
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
# Avoid cmake warning if CMP0071 is not set.
if (POLICY CMP0071)
    cmake_policy(SET CMP0071 NEW)
endif ()
#set_property(GLOBAL PROPERTY USE_FOLDERS ON)

#------------------------------------------------------------------------------

set(APP_LIBRARY app-library)
include(application_info.cmake)

# ------------------------------------------------------------------------------

set(TARGET_NAME linphone-qt)
set(CMAKE_CXX_STANDARD 11)

#-------------------------------------------------------------------------------
#				SET OPTIONS
#-------------------------------------------------------------------------------

option(ENABLE_UPDATE_CHECK "Enable update check." NO)
option(ENABLE_UNIT_TESTS "Enable unit test of SDK." NO  )
option(ENABLE_TESTS "Build with testing binaries of SDK" NO  )
option(ENABLE_TESTS_COMPONENTS "Build libbctoolbox-tester" NO  )
option(ENABLE_TOOLS "Enable tools of SDK" NO)
option(ENABLE_STRICT "Build with strict compilator flags e.g. -Wall -Werror" NO)

option(ENABLE_FFMPEG "Build mediastreamer2 with ffmpeg video support." YES)
option(ENABLE_OPUS "Build mediastreamer2 with the OPUS codec." YES)

if(WIN32 OR APPLE)
else()
    option(ENABLE_V4L "Ability to capture and display video using libv4l2 or libv4l." YES)
endif()

option(ENABLE_RELATIVE_PREFIX "Set Internal packages relative to the binary" YES)

include(GNUInstallDirs)
include(CheckCXXCompilerFlag)

find_package(LinphoneCxx CONFIG)
find_package(Linphone CONFIG)
find_package(bctoolbox CONFIG)
find_package(belcard CONFIG)
find_package(Mediastreamer2 CONFIG)
find_package(ortp CONFIG)

if( NOT (LinphoneCxx_FOUND) OR NOT (Linphone_FOUND) OR NOT (bctoolbox_FOUND) OR NOT (belcard_FOUND) OR NOT (Mediastreamer2_FOUND) OR NOT (ortp_FOUND))
    message("Linphone packages are not installed. Build and install it in order to build Minizip and then Desktop Application.")
    set(CMAKE_INSTALL_PREFIX "${LINPHONE_OUTPUT_DIR}")
    add_subdirectory("linphone-sdk" "${SDK_BUILD_DIR}")
elseif( APP_FORCE_SDK_BUILD )
    set(CMAKE_INSTALL_PREFIX "${LINPHONE_OUTPUT_DIR}")
    add_subdirectory("linphone-sdk" "${SDK_BUILD_DIR}")
else()
    # Linphone SDK has been builded
    message("LINPHONECXX : ${LINPHONECXX_INCLUDE_DIRS} => ${LINPHONECXX_LIBRARIES}")
    find_package(Minizip)
    if( NOT(Minizip_FOUND))
        message("Minizip are not installed. Build and install it in order to build Desktop Application.")
        set(CMAKE_INSTALL_PREFIX "${MINIZIP_OUTPUT_DIR}")
        add_subdirectory(submodules/externals/minizip "submodules/externals/minizip/build-minizip")
    else()		
        set(CMAKE_INSTALL_PREFIX "${APPLICATION_OUTPUT_DIR}")
		if(WIN32)
			set(EXECUTABLE_OUTPUT_DIR "${CMAKE_BINARY_DIR}")
			set( CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${EXECUTABLE_OUTPUT_DIR}")
			set( CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${EXECUTABLE_OUTPUT_DIR}")
			set( CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO "${EXECUTABLE_OUTPUT_DIR}")
		endif()
        message("MINIZIP : ${MINIZIP_INCLUDE_DIRS} => ${MINIZIP_LIBRARIES}")
        message("INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} FRAMEWORK_PATH=${CMAKE_FRAMEWORK_PATH}")
        message("LINPHONE : ${LINPHONE_INCLUDE_DIRS} => ${LINPHONE_LIBRARIES}")
        message("LINPHONECXX : ${LINPHONECXX_INCLUDE_DIRS} => ${LINPHONECXX_LIBRARIES}")

        # Build configuration
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -DNDEBUG -DQT_NO_DEBUG")
        set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -DQT_QML_DEBUG -DQT_DECLARATIVE_DEBUG")
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DDEBUG -DQT_QML_DEBUG -DQT_DECLARATIVE_DEBUG")
        if( WIN32)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_WINSOCKAPI_")#remove error from windows headers order
        endif()
        set(CMAKE_INCLUDE_CURRENT_DIR ON)#useful for config.h

        set(QT5_PACKAGES Core Gui Quick Widgets QuickControls2 Svg LinguistTools Concurrent Network)
        if (UNIX AND NOT APPLE)
            list(APPEND QT5_PACKAGES DBus)
        endif ()
        set(QT5_PACKAGES_OPTIONAL TextToSpeech)
        set(CMAKE_AUTOMOC ON)
        #-------------------------------------------------
        set(ASSETS_DIR "assets")
        set(SOURCES
            src/app/App.cpp
            src/app/AppController.cpp
            src/app/cli/Cli.cpp
            src/app/logger/Logger.cpp
            src/app/paths/Paths.cpp
            src/app/providers/AvatarProvider.cpp
            src/app/providers/ImageProvider.cpp
            src/app/providers/ThumbnailProvider.cpp
            src/app/translator/DefaultTranslator.cpp
            src/components/assistant/AssistantModel.cpp
            src/components/authentication/AuthenticationNotifier.cpp
            src/components/call/CallModel.cpp
            src/components/calls/CallsListModel.cpp
            src/components/calls/CallsListProxyModel.cpp
            src/components/camera/Camera.cpp
            src/components/camera/CameraPreview.cpp
            src/components/camera/MSFunctions.cpp
            src/components/chat/ChatModel.cpp
            src/components/chat/ChatProxyModel.cpp
            src/components/codecs/AbstractCodecsModel.cpp
            src/components/codecs/AudioCodecsModel.cpp
            src/components/codecs/VideoCodecsModel.cpp
            src/components/conference/ConferenceAddModel.cpp
            src/components/conference/ConferenceHelperModel.cpp
            src/components/conference/ConferenceModel.cpp
            src/components/contact/ContactModel.cpp
            src/components/contact/VcardModel.cpp
            src/components/contacts/ContactsListModel.cpp
            src/components/contacts/ContactsListProxyModel.cpp
            src/components/core/CoreHandlers.cpp
            src/components/core/CoreManager.cpp
            src/components/core/event-count-notifier/AbstractEventCountNotifier.cpp
            src/components/file/FileDownloader.cpp
            src/components/file/FileExtractor.cpp
            src/components/notifier/Notifier.cpp
            src/components/other/clipboard/Clipboard.cpp
            src/components/other/colors/Colors.cpp
            src/components/other/text-to-speech/TextToSpeech.cpp
            src/components/other/units/Units.cpp
            src/components/presence/OwnPresenceModel.cpp
            src/components/presence/Presence.cpp
            src/components/settings/AccountSettingsModel.cpp
            src/components/settings/SettingsModel.cpp
            src/components/sip-addresses/SipAddressesModel.cpp
            src/components/sip-addresses/SipAddressesProxyModel.cpp
            src/components/sip-addresses/SipAddressObserver.cpp
            src/components/sound-player/SoundPlayer.cpp
            src/components/telephone-numbers/TelephoneNumbersModel.cpp
            src/components/timeline/TimelineModel.cpp
            src/components/url-handlers/UrlHandlers.cpp
            src/utils/LinphoneUtils.cpp
            src/utils/MediastreamerUtils.cpp
            src/utils/QExifImageHeader.cpp
            src/utils/Utils.cpp
            )

        set(HEADERS
            src/app/App.hpp
            src/app/AppController.hpp
            src/app/cli/Cli.hpp
            src/app/logger/Logger.hpp
            src/app/paths/Paths.hpp
            src/app/providers/AvatarProvider.hpp
            src/app/providers/ImageProvider.hpp
            src/app/providers/ThumbnailProvider.hpp
            src/app/single-application/SingleApplication.hpp
            src/app/translator/DefaultTranslator.hpp
            src/components/assistant/AssistantModel.hpp
            src/components/authentication/AuthenticationNotifier.hpp
            src/components/call/CallModel.hpp
            src/components/calls/CallsListModel.hpp
            src/components/calls/CallsListProxyModel.hpp
            src/components/camera/Camera.hpp
            src/components/camera/CameraPreview.hpp
            src/components/camera/MSFunctions.hpp
            src/components/chat/ChatModel.hpp
            src/components/chat/ChatProxyModel.hpp
            src/components/codecs/AbstractCodecsModel.hpp
            src/components/codecs/AudioCodecsModel.hpp
            src/components/codecs/VideoCodecsModel.hpp
            src/components/Components.hpp
            src/components/conference/ConferenceAddModel.hpp
            src/components/conference/ConferenceHelperModel.hpp
            src/components/conference/ConferenceModel.hpp
            src/components/contact/ContactModel.hpp
            src/components/contact/VcardModel.hpp
            src/components/contacts/ContactsListModel.hpp
            src/components/contacts/ContactsListProxyModel.hpp
            src/components/core/CoreHandlers.hpp
            src/components/core/CoreManager.hpp
            src/components/core/event-count-notifier/AbstractEventCountNotifier.hpp
            src/components/file/FileDownloader.hpp
            src/components/file/FileExtractor.hpp
            src/components/notifier/Notifier.hpp
            src/components/other/clipboard/Clipboard.hpp
            src/components/other/colors/Colors.hpp
            src/components/other/desktop-tools/DesktopTools.hpp
            src/components/other/text-to-speech/TextToSpeech.hpp
            src/components/other/units/Units.hpp
            src/components/presence/OwnPresenceModel.hpp
            src/components/presence/Presence.hpp
            src/components/settings/AccountSettingsModel.hpp
            src/components/settings/SettingsModel.hpp
            src/components/sip-addresses/SipAddressesModel.hpp
            src/components/sip-addresses/SipAddressesProxyModel.hpp
            src/components/sip-addresses/SipAddressObserver.hpp
            src/components/sound-player/SoundPlayer.hpp
            src/components/telephone-numbers/TelephoneNumbersModel.hpp
            src/components/timeline/TimelineModel.hpp
            src/components/url-handlers/UrlHandlers.hpp
            src/utils/LinphoneUtils.hpp
            src/utils/MediastreamerUtils.hpp
            src/utils/QExifImageHeader.hpp
            src/utils/Utils.hpp
            )

        set(MAIN_FILE src/app/main.cpp)

        if (APPLE)
            list(APPEND SOURCES
            src/app/single-application/SingleApplication.cpp
            src/components/core/event-count-notifier/EventCountNotifierMacOs.m
            src/components/other/desktop-tools/DesktopToolsMacOs.cpp
            src/components/other/desktop-tools/screen-saver/ScreenSaverMacOs.m
            src/components/other/desktop-tools/state-process/StateProcessMacOs.mm
            )
            list(APPEND HEADERS
            src/app/single-application/SingleApplicationPrivate.hpp
            src/components/core/event-count-notifier/EventCountNotifierMacOs.hpp
            src/components/other/desktop-tools/DesktopToolsMacOs.hpp
            )
        elseif (WIN32)
            list(APPEND SOURCES
            src/app/single-application/SingleApplication.cpp
            src/components/core/event-count-notifier/EventCountNotifierSystemTrayIcon.cpp
            src/components/other/desktop-tools/DesktopToolsWindows.cpp
            )
            list(APPEND HEADERS
            src/app/single-application/SingleApplicationPrivate.hpp
            src/components/core/event-count-notifier/EventCountNotifierSystemTrayIcon.hpp
            src/components/other/desktop-tools/DesktopToolsWindows.hpp
            )
        else ()
            list(APPEND SOURCES
            src/app/single-application/SingleApplicationDBus.cpp
            src/components/core/event-count-notifier/EventCountNotifierSystemTrayIcon.cpp
            src/components/other/desktop-tools/DesktopToolsLinux.cpp
            src/components/other/desktop-tools/screen-saver/ScreenSaverDBus.cpp
            src/components/other/desktop-tools/screen-saver/ScreenSaverXdg.cpp
            )
            list(APPEND HEADERS
            src/app/single-application/SingleApplicationDBusPrivate.hpp
            src/components/core/event-count-notifier/EventCountNotifierSystemTrayIcon.hpp
            src/components/other/desktop-tools/DesktopToolsLinux.hpp
            src/components/other/desktop-tools/screen-saver/ScreenSaverDBus.hpp
            src/components/other/desktop-tools/screen-saver/ScreenSaverXdg.hpp
            )
        endif ()

        set(QRC_RESOURCES resources.qrc)

        set(LANGUAGES_DIRECTORY "${ASSETS_DIR}/languages")
        set(I18N_FILENAME i18n.qrc)
        set(LANGUAGES de en fr_FR ja lt ru sv tr pt_BR)

        # ------------------------------------------------------------------------------

        function (PREPEND list prefix)
            set(new_list "")

            foreach (elem ${${list}})
            list(APPEND new_list "${prefix}${elem}")
            endforeach ()

            set(${list} ${new_list} PARENT_SCOPE)
        endfunction ()

        # Force absolute paths.
        PREPEND(SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/")
        PREPEND(HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/")

        # ------------------------------------------------------------------------------
        # Compute QML files list.
        # ------------------------------------------------------------------------------

        set(QML_SOURCES)
        file(STRINGS ${QRC_RESOURCES} QRC_RESOURCES_CONTENT)
        foreach (line ${QRC_RESOURCES_CONTENT})
            set(result)
            string(REGEX REPLACE
            "^[ \t]*<[ \t]*file[ \t]*>[ \t]*(.+\\.[a-z]+)[ \t]*<[ \t]*/[ \t]*file[ \t]*>[ \t]*$"
            "\\1"
            result
            "${line}"
            )
            string(REGEX MATCH "\\.[a-z]+$" is_ui ${result})
            if (NOT ${is_ui} STREQUAL "")
            list(APPEND QML_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/${result}")
            endif ()
        endforeach ()

        # ------------------------------------------------------------------------------
        # Init git hooks.
        # ------------------------------------------------------------------------------

        if (NOT WIN32)
            add_custom_target(
            check_qml DEPENDS ${QML_SOURCES}
            COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/tools/check_qml_syntax"
            )
        endif()

        execute_process(COMMAND ${CMAKE_COMMAND} -E copy
            "${CMAKE_CURRENT_SOURCE_DIR}/tools/private/pre-commit"
            "${CMAKE_CURRENT_SOURCE_DIR}/.git/hooks/pre-commit"
            )
        set(_QML_IMPORT_PATHS "")
        list(APPEND _QML_IMPORT_PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ui/modules")
        list(APPEND _QML_IMPORT_PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ui/dev-modules")
        list(APPEND _QML_IMPORT_PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ui/scripts")
        list(APPEND _QML_IMPORT_PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ui/views")


        set(QML_IMPORT_PATH ${_QML_IMPORT_PATHS} CACHE STRING "Path used to locate CMake modules by Qt Creator" FORCE)


        if(APPLE)
            if(MS2_PLUGINS_LOCATION)
            set(MSPLUGINS_DIR ${MS2_PLUGINS_LOCATION})
            else()
            set(MSPLUGINS_DIR "Frameworks/mediastreamer2.framework/Versions/A/Libraries")
            endif()
        else()
            set(MSPLUGINS_DIR "${CMAKE_INSTALL_LIBDIR}/mediastreamer/plugins")
        endif()
        configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/config.h.cmake" "${CMAKE_CURRENT_BINARY_DIR}/config.h")
        # ------------------------------------------------------------------------------
        # Build.
        # ------------------------------------------------------------------------------

        include_directories(src/)

        find_package(Qt5 COMPONENTS ${QT5_PACKAGES} REQUIRED)
        find_package(Qt5 COMPONENTS ${QT5_PACKAGES_OPTIONAL} QUIET)

        if (CMAKE_INSTALL_RPATH)
            #Retrieve lib path from a know QT executable
            get_target_property(LUPDATE_PATH Qt5::lupdate LOCATION)
            get_filename_component(LUPDATE_PATH "${LUPDATE_PATH}" DIRECTORY)
            get_filename_component(QT_PATH "${LUPDATE_PATH}/../lib" ABSOLUTE)
            list(APPEND CMAKE_INSTALL_RPATH "${QT_PATH}")
        endif ()
        # Add languages support.
        add_subdirectory(${LANGUAGES_DIRECTORY})

        list(APPEND SOURCES "${CMAKE_CURRENT_BINARY_DIR}/${LANGUAGES_DIRECTORY}/${I18N_FILENAME}")

        # Build.
        #add_library(${APP_LIBRARY} OBJECT ${SOURCES} ${HEADERS} ${RESOURCES} ${QML_SOURCES})
        add_library(${APP_LIBRARY} OBJECT ${SOURCES} ${HEADERS} ${QML_SOURCES} ${QRC_RESOURCES})
        set_property(TARGET ${APP_LIBRARY} PROPERTY POSITION_INDEPENDENT_CODE ON)

        #Turn on automatic resources compilation by cmake
        #Instead of excplicitely calling qt5_add_resources
        set_property(TARGET ${APP_LIBRARY} PROPERTY AUTORCC ON)

        bc_git_version(${TARGET_NAME} ${PROJECT_VERSION})
        add_dependencies(${APP_LIBRARY} ${TARGET_NAME}-git-version)
        add_dependencies(${APP_LIBRARY} update_translations)
        if (WIN32)
            add_executable(${TARGET_NAME} WIN32 $<TARGET_OBJECTS:${APP_LIBRARY}> ${ASSETS_DIR}/app-icon.rc ${MAIN_FILE})
        else ()
            add_executable(${TARGET_NAME} $<TARGET_OBJECTS:${APP_LIBRARY}> ${MAIN_FILE})
        endif ()

        set_target_properties(${TARGET_NAME} PROPERTIES OUTPUT_NAME "${EXECUTABLE_NAME}")
        set(INCLUDED_DIRECTORIES "${LINPHONECXX_INCLUDE_DIRS}" "${LINPHONE_INCLUDE_DIRS}" "${BELCARD_INCLUDE_DIRS}" "${BCTOOLBOX_INCLUDE_DIRS}" "${MEDIASTREAMER2_INCLUDE_DIRS}" "${MINIZIP_INCLUDE_DIRS}")
        set(LIBRARIES ${BCTOOLBOX_CORE_LIBRARIES} ${BELCARD_LIBRARIES}  ${LINPHONE_LIBRARIES} ${LINPHONECXX_LIBRARIES} ${MEDIASTREAMER2_LIBRARIES} ${ORTP_LIBRARIES} ${OPUS_LIBRARIES})
        if(WIN32)
            list(APPEND LIBRARIES ${MINIZIP_LIBRARIES})
        endif()
        foreach (package ${QT5_PACKAGES})
            list(APPEND INCLUDED_DIRECTORIES "${Qt5${package}_INCLUDE_DIRS}")
            # `qt5_create_translation` is provided from `LinguistTools` package.
            # But the `Qt5::LinguistTools` lib does not exist. Remove it.
            if (NOT (${package} STREQUAL LinguistTools))
            list(APPEND LIBRARIES ${Qt5${package}_LIBRARIES})
            endif ()
        endforeach ()

        foreach (package ${QT5_PACKAGES_OPTIONAL})
            if ("${Qt5${package}_FOUND}")
            message("Optional package ${package} found.")
            list(APPEND INCLUDED_DIRECTORIES "${Qt5${package}_INCLUDE_DIRS}")
            list(APPEND LIBRARIES ${Qt5${package}_LIBRARIES})

            string(TOUPPER "${package}" INCLUDE_NAME)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D${INCLUDE_NAME}_ENABLED")
            else ()
            message("Optional package ${package} not found.")
            endif ()
        endforeach ()

        if (APPLE)
            list(APPEND LIBRARIES "-framework Cocoa -framework IOKit")
            # -framework linphone") #This doesn't work yet
        endif ()
        target_include_directories(${APP_LIBRARY} SYSTEM PRIVATE ${INCLUDED_DIRECTORIES})
        target_include_directories(${TARGET_NAME} SYSTEM PRIVATE ${INCLUDED_DIRECTORIES})

        target_link_libraries(${TARGET_NAME} ${LIBRARIES})
        if(WIN32)
            target_link_libraries(${TARGET_NAME} wsock32 ws2_32)
        endif()

        foreach (target ${TARGET_NAME})
            message("Available target : ${target}")
    #	    install(TARGETS ${target}
    #		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    #		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    #		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    #		PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
    #		)
        endforeach ()

        if (UNIX AND NOT APPLE)
            # Install desktop/icon files.
            configure_file("${ASSETS_DIR}/linphone.desktop.cmake" "${EXECUTABLE_NAME}.desktop" @ONLY)

            install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${EXECUTABLE_NAME}.desktop"
            DESTINATION "${CMAKE_INSTALL_DATADIR}/applications"
            )
            install(FILES "${ASSETS_DIR}/images/linphone_logo.svg"
            DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/scalable/apps/"
            RENAME "${EXECUTABLE_NAME}.svg"
            )

            set(ICON_DIRS 16x16 22x22 24x24 32x32 64x64 128x128 256x256)
            foreach (DIR ${ICON_DIRS})
            install(FILES "${ASSETS_DIR}/icons/hicolor/${DIR}/apps/icon.png"
                DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/${DIR}/apps/"
                RENAME "${EXECUTABLE_NAME}.png"
                )
            endforeach ()
        endif ()

        # ------------------------------------------------------------------------------
        # CPack settings & RPM.
        # ------------------------------------------------------------------------------
        set(LINPHONE_DESKTOP_DIR "${CMAKE_SOURCE_DIR}")
        set(LINPHONE_SDK_DIR "${CMAKE_BINARY_DIR}/linphone-sdk")
        set(TOOLS_DIR "${CMAKE_BINARY_DIR}/programs")
        set(LINPHONE_SOURCE_DIR ${EP_linphone_SOURCE_DIR})
        set(LINPHONE_BUILDER_SIGNING_IDENTITY ${LINPHONE_BUILDER_SIGNING_IDENTITY})
        set(LINPHONE_SDK_TMP "${CMAKE_CURRENT_BINARY_DIR}/${EXECUTABLE_NAME}-sdk-tmp")

        add_subdirectory(build)
        add_subdirectory(cmake_builder/linphone_package)

        # ------------------------------------------------------------------------------
        # To start better integration into IDE.
        # ------------------------------------------------------------------------------

        source_group(
            "Qml" REGULAR_EXPRESSION ".+\.qml$"
            )
        source_group(
            "Js" REGULAR_EXPRESSION ".+\.js$"
            )
        source_group(
            "Svg" REGULAR_EXPRESSION ".+\.svg$"
            )

        # ------------------------------
        # Installation
        # ------------------------------
        if(APPLE)
            target_link_libraries(${TARGET_NAME} "${MINIZIP_OUTPUT_DIR}/${CMAKE_INSTALL_LIBDIR}/libminizip.dylib")
            execute_process(COMMAND install_name_tool -id "@executable_path/../Frameworks/libminizip.dylib" "${MINIZIP_OUTPUT_DIR}/${CMAKE_INSTALL_LIBDIR}/libminizip.dylib")
        elseif(NOT WIN32)
            target_link_libraries(${TARGET_NAME} "${MINIZIP_OUTPUT_DIR}/${CMAKE_INSTALL_LIBDIR}/libminizip.so")
        endif()
    endif()#If (LinphoneCxx_FOUND AND Minizip_FOUND)
endif()#If (LinphoneCxx_FOUND)
