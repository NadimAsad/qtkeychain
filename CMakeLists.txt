cmake_minimum_required(VERSION 3.19)

set(QTKEYCHAIN_VERSION 0.12.90)
set(QTKEYCHAIN_SOVERSION 1)

project(qtkeychain
    VERSION ${QTKEYCHAIN_VERSION}
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

option(BUILD_TEST_APPLICATION "Build test application" OFF)
option(BUILD_TRANSLATIONS "Build translations" ON)

if(CMAKE_SYSTEM_NAME STREQUAL Android)
    set(ANDROID 1)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL Haiku)
    set(HAIKU 1)
endif()

if (WIN32)
    option(USE_CREDENTIAL_STORE "Build with windows CredentialStore support" ON)

    if (USE_CREDENTIAL_STORE)
        add_definitions(-DUSE_CREDENTIAL_STORE=1)
    endif()
endif()

find_package(Qt5 COMPONENTS Core REQUIRED)

if(ANDROID)
    find_package(Qt5 COMPONENTS AndroidExtras REQUIRED)
endif()

if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    find_package(Qt5 5.15 COMPONENTS DBus REQUIRED)
endif()

if(BUILD_TRANSLATIONS)
    find_package(Qt5 COMPONENTS LinguistTools REQUIRED)
endif()

set(qtkeychain_SOURCES
    source/keychain.cpp
)

if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    if (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    endif()
else()
    add_definitions( -Wall )
endif()

if(WIN32)
    list(APPEND qtkeychain_SOURCES
        source/keychain_win.cpp
    )

    if (NOT USE_CREDENTIAL_STORE)
        list(APPEND qtkeychain_SOURCES
            source/plaintextstore.cpp
        )
    endif()

    #FIXME: mingw bug; otherwise getting undefined refs to RtlSecureZeroMemory there
    if(MINGW)
        add_definitions( -O2 )
    endif()
endif()

if(APPLE)
    list(APPEND qtkeychain_SOURCES
        source/keychain_apple.mm
    )
endif()

if(HAIKU)
    list(APPEND qtkeychain_SOURCES
        source/keychain_haiku.cpp
    )

    find_library(BE_LIBRARY be REQUIRED)
endif()

if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    option(LIBSECRET_SUPPORT "Build with libsecret support" ON)

    if(LIBSECRET_SUPPORT)
        include(FindPkgConfig)
        pkg_check_modules(LIBSECRET REQUIRED libsecret-1)
        add_definitions(-DHAVE_LIBSECRET=1)
    endif()

    add_definitions(-DKEYCHAIN_DBUS=1)
    list(APPEND qtkeychain_SOURCES
        source/keychain_unix.cpp
        source/gnomekeyring.cpp
        source/libsecret.cpp
        source/plaintextstore.cpp
    )

    qt_add_dbus_interface(qtkeychain_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/source/org.kde.KWallet.xml kwallet_interface KWalletInterface)

    qt_wrap_cpp(qtkeychain_MOC
        private/gnomekeyring_p.h
    )
endif()

if(ANDROID)
    list(APPEND qtkeychain_SOURCES
        source/keychain_android.cpp
        source/androidkeystore.cpp
        source/plaintextstore.cpp
    )
endif()

qt_wrap_cpp(qtkeychain_MOC
    include/keychain.h
    private/keychain_p.h
)

set(qtkeychain_TR_FILES
    translations/qtkeychain_de.ts
    translations/qtkeychain_fr.ts
    translations/qtkeychain_ro.ts
    translations/qtkeychain_ru.ts
    translations/qtkeychain_zh.ts
)

file(GLOB qtkeychain_TR_SOURCES source/*.cpp include/*.h)
if (BUILD_TRANSLATION)
    qt_create_translation(qtkeychain_MESSAGES ${qtkeychain_TR_SOURCES} ${qtkeychain_TR_FILES})
    qt_add_translation(qtkeychain_QM_FILES ${qtkeychain_TR_FILES})
    add_custom_target(messages DEPENDS ${qtkeychain_MESSAGES})
    add_custom_target(translations DEPENDS ${qtkeychain_QM_FILES} messages)
endif()

add_library(${PROJECT_NAME}
    ${qtkeychain_MOC}
    ${qtkeychain_SOURCES}
    ${qtkeychain_QM_FILES}
)
if(BUILD_TRANSLATION)
    add_dependencies(${PROJECT_NAME} translations)
endif()

target_link_libraries(${PROJECT_NAME}
    PUBLIC
        Qt5::Core
)

if(WIN32)
    target_link_libraries(${PROJECT_NAME}
        PUBLIC 
            crypt32
    )

    set_target_properties(${PROJECT_NAME}
        PROPERTIES
            DEBUG_POSTFIX "d"
    )
endif()

if(ANDROID)
    target_link_libraries(${PROJECT_NAME}
        PUBLIC 
            Qt5::AndroidExtras
    )
endif()

if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    if(LIBSECRET_SUPPORT)
        target_include_directories(${PROJECT_NAME}
            PUBLIC
                ${LIBSECRET_INCLUDE_DIRS}
        )

        target_link_libraries(${PROJECT_NAME}
            PUBLIC
                ${LIBSECRET_LIBRARY_DIRS}
            PRIVATE
                ${LIBSECRET_LIBRARIES}
        )
    endif()

    target_link_libraries(${PROJECT_NAME}
        PUBLIC
            Qt5::DBus
    )
endif()

if(APPLE)
    target_link_libraries(${PROJECT_NAME}
        PUBLIC
            "-framework Foundation"
            "-framework Security"
    )
endif()

if(HAIKU)
    target_link_libraries(${PROJECT_NAME}
        PUBLIC
            ${BE_LIBRARY}
    )
endif()

target_include_directories(${PROJECT_NAME}
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${CMAKE_CURRENT_BINARY_DIR}
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/private
)

include(GenerateExportHeader)
generate_export_header(${PROJECT_NAME}
    EXPORT_FILE_NAME qkeychain_export.h
    EXPORT_MACRO_NAME QKEYCHAIN_EXPORT
)

set_target_properties(${PROJECT_NAME} PROPERTIES
    VERSION ${QTKEYCHAIN_VERSION}
    SOVERSION ${QTKEYCHAIN_SOVERSION}
    MACOSX_RPATH 1
)

if(BUILD_TEST_APPLICATION)
    add_executable(testclient
        source/testclient.cpp
    )

    target_link_libraries(testclient
        ${PROJECT_NAME}
    )
endif()
