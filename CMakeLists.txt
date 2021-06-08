cmake_minimum_required(VERSION 3.19)

set(QTKEYCHAIN_VERSION 0.12.90)
set(QTKEYCHAIN_SOVERSION 1)

project(qtkeychain VERSION ${QTKEYCHAIN_VERSION} LANGUAGES CXX)

# Enable C++11
SET(CMAKE_CXX_STANDARD 11)

option(BUILD_WITH_QT6 "Build qtkeychain with Qt 6" OFF)
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

if( NOT BUILD_WITH_QT6 )
    find_package(Qt5 COMPONENTS Core REQUIRED)
endif()

if (Qt5Core_FOUND AND NOT BUILD_WITH_QT6)
  set(QTKEYCHAIN_VERSION_INFIX 5)

  if(ANDROID)
    if(Qt5Core_VERSION VERSION_LESS 5.7)
        find_package(Qt5 COMPONENTS Core REQUIRED Private)
        include_directories(${Qt5Core_PRIVATE_INCLUDE_DIRS})
    endif()

    find_package(Qt5 COMPONENTS AndroidExtras REQUIRED)
    include_directories(${Qt5AndroidExtras_INCLUDE_DIRS})
    set(QTANDROIDEXTRAS_LIBRARIES ${Qt5AndroidExtras_LIBRARIES})
  endif()

  if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    find_package(Qt5 COMPONENTS DBus REQUIRED)
    include_directories(${Qt5DBus_INCLUDE_DIRS})
    set(QTDBUS_LIBRARIES ${Qt5DBus_LIBRARIES})
  endif()

  if(BUILD_TRANSLATIONS)
    find_package(Qt5 COMPONENTS LinguistTools REQUIRED)
  endif()

  set(QTCORE_LIBRARIES ${Qt5Core_LIBRARIES})
  include_directories(${Qt5Core_INCLUDE_DIRS})
else()
  find_package(Qt6 COMPONENTS Core REQUIRED)
  set(QTKEYCHAIN_VERSION_INFIX 6)

  if(ANDROID)
    find_package(Qt6 COMPONENTS AndroidExtras REQUIRED)
    include_directories(${Qt6AndroidExtras_INCLUDE_DIRS})
    set(QTANDROIDEXTRAS_LIBRARIES ${Qt6AndroidExtras_LIBRARIES})
  endif()

  if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    find_package(Qt6 COMPONENTS DBus REQUIRED)
    include_directories(${Qt6DBus_INCLUDE_DIRS})
    set(QTDBUS_LIBRARIES ${Qt6DBus_LIBRARIES})
  endif()

  if(BUILD_TRANSLATIONS)
    find_package(Qt6 COMPONENTS LinguistTools REQUIRED)
  endif()

  set(QTCORE_LIBRARIES ${Qt6Core_LIBRARIES})
endif()


list(APPEND qtkeychain_LIBRARIES ${QTCORE_LIBRARIES})
set(qtkeychain_SOURCES
    source/keychain.cpp
)

if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # CMake < 3.15 sneaks in /W# flags for us, so we need a replacement,
    # or we'll get a warning (cf. CMP0092)
    if (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    endif()
else()
    # MSVC's STL / Qt headers are not MSVC -Wall clean, so don't enable it there
    add_definitions( -Wall )
endif()

if(WIN32)
    list(APPEND qtkeychain_SOURCES source/keychain_win.cpp)
    if (NOT USE_CREDENTIAL_STORE)
        list(APPEND qtkeychain_LIBRARIES crypt32)
        list(APPEND qtkeychain_SOURCES source/plaintextstore.cpp)
    endif()
    #FIXME: mingw bug; otherwise getting undefined refs to RtlSecureZeroMemory there
    if(MINGW)
        add_definitions( -O2 )
    endif()
endif()

if(APPLE)
    list(APPEND qtkeychain_SOURCES source/keychain_apple.mm)
    list(APPEND qtkeychain_LIBRARIES "-framework Foundation" "-framework Security")
endif()

if(HAIKU)
    list(APPEND qtkeychain_SOURCES source/keychain_haiku.cpp)

    find_library(BE_LIBRARY be REQUIRED)
    list(APPEND qtkeychain_LIBRARIES ${BE_LIBRARY})
endif()

if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    option(LIBSECRET_SUPPORT "Build with libsecret support" ON)

    if(LIBSECRET_SUPPORT)
        include(FindPkgConfig)
        pkg_check_modules(LIBSECRET REQUIRED libsecret-1)
        add_definitions(-DHAVE_LIBSECRET=1)
        INCLUDE_DIRECTORIES(${LIBSECRET_INCLUDE_DIRS})
        LINK_DIRECTORIES(${LIBSECRET_LIBRARY_DIRS})
        list(APPEND qtkeychain_LIBRARIES_PRIVATE ${LIBSECRET_LIBRARIES})
    endif()

    add_definitions(-DKEYCHAIN_DBUS=1)
    list(APPEND qtkeychain_SOURCES source/keychain_unix.cpp source/gnomekeyring.cpp source/libsecret.cpp source/plaintextstore.cpp)
    qt_add_dbus_interface(qtkeychain_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/source/org.kde.KWallet.xml kwallet_interface KWalletInterface)
    list(APPEND qtkeychain_LIBRARIES ${QTDBUS_LIBRARIES} )
endif()

if(ANDROID)
    list(APPEND qtkeychain_SOURCES source/keychain_android.cpp source/androidkeystore.cpp source/plaintextstore.cpp)
    list(APPEND qtkeychain_LIBRARIES_PRIVATE ${QTANDROIDEXTRAS_LIBRARIES} )
endif()

qt_wrap_cpp(qtkeychain_MOC_OUTFILES include/keychain.h private/keychain_p.h private/gnomekeyring_p.h)

set(qtkeychain_TR_FILES
      translations/qtkeychain_de.ts
      translations/qtkeychain_fr.ts
      translations/qtkeychain_ro.ts
      translations/qtkeychain_ru.ts
      translations/qtkeychain_zh.ts
)

set(QTKEYCHAIN_TARGET_NAME qt${QTKEYCHAIN_VERSION_INFIX}keychain)
add_library(${QTKEYCHAIN_TARGET_NAME} ${qtkeychain_SOURCES} ${qtkeychain_MOC_OUTFILES} ${qtkeychain_QM_FILES})

if(WIN32)
    set_target_properties( ${QTKEYCHAIN_TARGET_NAME} PROPERTIES DEBUG_POSTFIX "d" )
endif()

file(GLOB qtkeychain_TR_SOURCES *.cpp *.h)
if ( BUILD_TRANSLATIONS )
    qt_create_translation(qtkeychain_MESSAGES ${qtkeychain_TR_SOURCES} ${qtkeychain_TR_FILES})
    qt_add_translation(qtkeychain_QM_FILES ${qtkeychain_TR_FILES})
    add_custom_target(messages DEPENDS ${qtkeychain_MESSAGES})
    add_custom_target(translations DEPENDS ${qtkeychain_QM_FILES} messages)
    add_dependencies(${QTKEYCHAIN_TARGET_NAME} translations)
endif( BUILD_TRANSLATIONS )

target_link_libraries(${QTKEYCHAIN_TARGET_NAME} PUBLIC ${qtkeychain_LIBRARIES} PRIVATE ${qtkeychain_LIBRARIES_PRIVATE})
target_include_directories(${QTKEYCHAIN_TARGET_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/>
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/private>
)

include(GenerateExportHeader)
generate_export_header(${QTKEYCHAIN_TARGET_NAME}
  EXPORT_FILE_NAME qkeychain_export.h
  EXPORT_MACRO_NAME QKEYCHAIN_EXPORT
)

set_target_properties(${QTKEYCHAIN_TARGET_NAME} PROPERTIES
    VERSION ${QTKEYCHAIN_VERSION}
    SOVERSION ${QTKEYCHAIN_SOVERSION}
    MACOSX_RPATH 1
)

if(BUILD_TEST_APPLICATION)
    add_executable( testclient source/testclient.cpp )
    target_link_libraries( testclient ${QTKEYCHAIN_TARGET_NAME})
endif()
