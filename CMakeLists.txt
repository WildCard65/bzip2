cmake_minimum_required(VERSION 3.15)
if(COMMAND cmake_policy)
    cmake_policy(SET CMP0048 NEW)
    cmake_policy(SET CMP0091 NEW)
endif()

file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/bzlib.h.cmakein" BZLIB_H REGEX "bzip2/libbzip2 version [0-9]+\\.[^ ]+ of [0-9]+ ")
string(REGEX REPLACE ".* bzip2/libbzip2 version ([0-9]+\\.[^ ]+) of [0-9]+ .*" "\\1" BZIP2_VERSION_STRING "${BZLIB_H}")
unset(BZLIB_H)

message(STATUS "Configuring bzip2 version ${BZIP2_VERSION_STRING}")
project(bzip2 VERSION ${BZIP2_VERSION_STRING} LANGUAGES C)

# Enable the ability to use the static MSVC C runtime if targeting Windows.
if(WIN32)
    option(BZIP2_USE_STATIC_MSVCRT "Link bzip2 against the static MSVC C runtime" OFF)
endif()

set(BZIP2_LIBRARY_NAME "" CACHE STRING "The name of bzip2's library (Defaults to: bz2)")
if("${BZIP2_LIBRARY_NAME}" STREQUAL "")
    set(_BZIP2_LIBRARY_NAME "bz2" CACHE INTERNAL "")
else()
    set(_BZIP2_LIBRARY_NAME "${BZIP2_LIBRARY_NAME}" CACHE INTERNAL "")
endif()

option(BZIP2_NO_STDIO "Build bzip2 without \"stdio.h\"" OFF)
### Only allow the BZ_USE_WINAPI macro on the Windows platform ###
if(WIN32)
    option(BZIP2_USE_WINAPI "Build bzip2 with the \"BZ_USE_WINAPI\" macro" OFF)
endif()

include(GNUInstallDirs)
### Hack in GNUInstallDirs compatible variables ###
_GNUInstallDirs_cache_path_fallback(CMAKE_INSTALL_PKGCONFIGDIR "${CMAKE_INSTALL_DATAROOTDIR}/pkgconfig" "pkgconfig (.pc) files")
_GNUInstallDirs_cache_path_fallback(CMAKE_INSTALL_CMAKEDIR "${CMAKE_INSTALL_DATAROOTDIR}/cmake" "CMake (.cmake) files")
mark_as_advanced(CMAKE_INSTALL_PKGCONFIGDIR CMAKE_INSTALL_CMAKEDIR)

GNUInstallDirs_get_absolute_install_dir(CMAKE_INSTALL_FULL_PKGCONFIGDIR CMAKE_INSTALL_PKGCONFIGDIR PKGCONFIG)
GNUInstallDirs_get_absolute_install_dir(CMAKE_INSTALL_FULL_CMAKEDIR CMAKE_INSTALL_CMAKEDIR CMAKEDIR)

macro(_BZIP2_MAP_CONFIG_OPTION option macro)
    set(${macro} ${${option}} CACHE INTERNAL "bzip2 C macro \"${macro}\"")
endmacro()

_BZIP2_MAP_CONFIG_OPTION(BZIP2_NO_STDIO BZ_NO_STDIO)
if(WIN32)
    _BZIP2_MAP_CONFIG_OPTION(BZIP2_USE_WINAPI BZ_USE_WINAPI)
endif()

if(MSVC)
    set(CMAKE_DEBUG_POSTFIX "d")
endif()

if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    # If we're doing an out of source build and the user has a bzlib.h
    # in their source tree...
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/bzlib.h)
        message(STATUS "Renaming")
        message(STATUS "    ${CMAKE_CURRENT_SOURCE_DIR}/bzlib.h")
        message(STATUS "to 'bzlib.h.included' because this file is included with bzip2")
        message(STATUS "but CMake generates it automatically in the build directory.")
        file(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/bzlib.h ${CMAKE_CURRENT_SOURCE_DIR}/bzlib.h.included)
    endif()
endif()

set(_BZIP2_PC ${CMAKE_CURRENT_BINARY_DIR}/bzip2.pc)
configure_file(${PROJECT_SOURCE_DIR}/bzip2.pc.cmakein ${_BZIP2_PC} @ONLY)
configure_file(${PROJECT_SOURCE_DIR}/bzlib.h.cmakein ${CMAKE_CURRENT_BINARY_DIR}/bzlib.h @ONLY)

set(_BZIP2_PUBLIC_HDRS ${CMAKE_CURRENT_BINARY_DIR}/bzlib.h)

set(BZIP2_NEED_PREFIX TRUE)
set(BZIP2_INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR})

if(MSVC)
    set_source_files_properties(compress.c PROPERTIES COMPILE_FLAGS -wd4244)
endif()

### Unlike what I did for zlib's CMake rewrite, my understanding is that bzip2 is meant for one or the other. - WildCard65 ###
add_library(bz2 blocksort.c huffman.c crctable.c randtable.c compress.c decompress.c bzlib.c ${_BZIP2_PUBLIC_HDRS} bzlib_private.h)
target_compile_definitions(bz2
    PRIVATE BZ_INTERNAL "$<$<CONFIG:Debug>:BZ_DEBUG>" "$<$<BOOL:${WIN32}>:_CRT_SECURE_NO_DEPRECATE>" "$<$<BOOL:${WIN32}>:_CRT_NONSTDC_NO_DEPRECATE>")
target_include_directories(bz2 PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_FULL_INCLUDEDIR}>)
set_target_properties(bz2 PROPERTIES
    MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>$<$<NOT:$<BOOL:${BZIP2_USE_STATIC_MSVCRT}>>:DLL>"
    OUTPUT_NAME "${_BZIP2_LIBRARY_NAME}"
    VERSION ${PROJECT_VERSION})

if(BUILD_SHARED_LIBS)
    target_compile_definitions(bz2 PUBLIC BZ_DLL)
    set_target_properties(bz2 PROPERTIES SOVERSION ${PROJECT_VERSION_MAJOR})
endif()

set(BZIP2_LIBRARIES "$<TARGET_FILE:bz2>")

if(UNIX AND BUILD_SHARED_LIBS)
    if(NOT APPLE)
        target_link_libraries(bz2 -lc)
    endif()
endif()

macro(_BZIP2_CREATE_EXECUTABLE tgt_name src)
    add_executable(${tgt_name} ${src})
    target_compile_definitions(${tgt_name} PRIVATE "$<$<BOOL:${WIN32}>:_CRT_SECURE_NO_DEPRECATE>" "$<$<BOOL:${WIN32}>:_CRT_NONSTDC_NO_DEPRECATE>")
    set_target_properties(${tgt_name} PROPERTIES
        MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>$<$<NOT:$<BOOL:${BZIP2_USE_STATIC_MSVCRT}>>:DLL>"
        VERSION ${PROJECT_VERSION})
endmacro()

if(NOT BUILD_SHARED_LIBS AND NOT BZIP2_NO_EXECUTABLES)
    _BZIP2_CREATE_EXECUTABLE(bzip2 bzip2.c)
    target_link_libraries(bzip2 PRIVATE bz2)

    _BZIP2_CREATE_EXECUTABLE(bzip2recover bzip2recover.c)
endif()

add_library(BZip2::BZip2 ALIAS bz2)
set(BZIP2_FOUND TRUE)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(TARGETS bz2
        EXPORT BZip2Target
        RUNTIME DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}"
        ARCHIVE DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}"
        LIBRARY DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}")
    install(EXPORT BZip2Target
        NAMESPACE BZip2::
        DESTINATION ${CMAKE_INSTALL_FULL_CMAKEDIR}
        FILE BZip2Targets.cmake)

    include(CMakePackageConfigHelpers)

    write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/BZip2ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion)
    configure_package_config_file(${PROJECT_SOURCE_DIR}/BZip2Config.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/BZip2Config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_FULL_CMAKEDIR}
        NO_SET_AND_CHECK_MACRO
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
        PATH_VARS CMAKE_INSTALL_INCLUDEDIR)

    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/BZip2Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/BZip2ConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_FULL_CMAKEDIR})
endif()

if(NOT BUILD_SHARED_LIBS AND TARGET bzip2 AND NOT SKIP_INSTALL_PROGRAMS AND NOT SKIP_INSTALL_ALL)
    install(TARGETS bzip2 bzip2recover RUNTIME DESTINATION ${CMAKE_INSTALL_FULL_BINDIR}
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
    install(FILES "$<TARGET_FILE:bzip2>" DESTINATION "${CMAKE_INSTALL_FULL_BINDIR}" RENAME "bunzip2$<$<BOOL:${WIN32}>:.exe>"
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_READ WORLD_EXECUTE)
    install(FILES "$<TARGET_FILE:bzip2>" DESTINATION "${CMAKE_INSTALL_FULL_BINDIR}" RENAME "bzcat$<$<BOOL:${WIN32}>:.exe>"
        PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

    if(UNIX)
        install(FILES bzgrep bzdiff bzmore DESTINATION "${CMAKE_INSTALL_FULL_BINDIR}"
            PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
    endif()
endif()

if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(FILES ${_BZIP2_PUBLIC_HDRS} DESTINATION "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
endif()

if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL)
    install(FILES ${_BZIP2_PC} DESTINATION "${CMAKE_INSTALL_FULL_PKGCONFIGDIR}")

    if(NOT SKIP_INSTALL_PROGRAMS)
        if(TARGET bzip2)
            install(FILES bzip2.1 DESTINATION "${CMAKE_INSTALL_FULL_MANDIR}/man1")
        endif()

        if(UNIX)
            file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/bzegrep.1 ".so man1/bzgrep.1")
            file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/bzfgrep.1 ".so man1/bzgrep.1")
            file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/bzless.1 ".so man1/bzmore.1")
            file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/bzcmp.1 ".so man1/bzdiff.1")

            install(FILES bzgrep.1 bzmore.1 bzdiff.1
                ${CMAKE_CURRENT_BINARY_DIR}/bzegrep.1
                ${CMAKE_CURRENT_BINARY_DIR}/bzfgrep.1
                ${CMAKE_CURRENT_BINARY_DIR}/bzless.1
                ${CMAKE_CURRENT_BINARY_DIR}/bzcmp.1
                DESTINATION "${CMAKE_INSTALL_FULL_MANDIR}/man1")
        endif()
    endif()
endif()
