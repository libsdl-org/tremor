cmake_minimum_required(VERSION 3.0)

# Extract project version from configure.ac
file(READ configure.ac CONFIGURE_AC_CONTENTS)
string(REGEX MATCH "AM_INIT_AUTOMAKE\\(libvorbisidec,([0-9]*\\.[0-9]*\\.[0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})

project(tremor VERSION ${CMAKE_MATCH_1} LANGUAGES C)

# Required modules
include(GNUInstallDirs)
include(CheckIncludeFiles)
include(CMakePackageConfigHelpers)
include(CTest)

# Build options
option(BUILD_SHARED_LIBS "Build shared library" OFF)
if(APPLE)
    option(BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif()
option(BUILD_LOW_ACCURACY "Enable 32 bit only multiply operations" OFF)

# Install options
option(INSTALL_DOCS "Install documentation" ON)
option(INSTALL_PKG_CONFIG_MODULE "Install vorbisidec.pc file" ON)
option(INSTALL_CMAKE_PACKAGE_MODULE "Install CMake package configuration module" ON)

# Extract library version from configure.ac
string(REGEX MATCH "LIB_CURRENT=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
set(LIB_CURRENT ${CMAKE_MATCH_1})

string(REGEX MATCH "LIB_AGE=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
set(LIB_AGE ${CMAKE_MATCH_1})

string(REGEX MATCH "LIB_REVISION=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
set(LIB_REVISION ${CMAKE_MATCH_1})

math(EXPR LIB_SOVERSION "${LIB_CURRENT} - ${LIB_AGE}")
set(LIB_VERSION "${LIB_SOVERSION}.${LIB_AGE}.${LIB_REVISION}")


# Helper function to configure pkg-config files
function(configure_pkg_config_file pkg_config_file_in)
    set(prefix ${CMAKE_INSTALL_PREFIX})
    set(exec_prefix ${CMAKE_INSTALL_FULL_BINDIR})
    set(libdir ${CMAKE_INSTALL_FULL_LIBDIR})
    set(includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
    set(VERSION ${PROJECT_VERSION})
    string(REPLACE ".in" "" pkg_config_file ${pkg_config_file_in})
    configure_file(${pkg_config_file_in} ${pkg_config_file} @ONLY)
endfunction()

message(STATUS "Configuring ${PROJECT_NAME} ${PROJECT_VERSION}")

# Find ogg dependency
if(NOT TARGET Ogg::ogg)
    find_package(Ogg REQUIRED)
endif()

include(CheckIncludeFile)
check_include_file(alloca.h HAVE_ALLOCA_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)

include(CheckSymbolExists)
if(HAVE_ALLOCA_H)
  check_symbol_exists(alloca "alloca.h" HAVE_ALLOCA)
else()
  check_symbol_exists(alloca "stdlib.h;malloc.h" HAVE_ALLOCA)
endif()
check_symbol_exists(__arm__    "" HAVE_ARM_ASM)

include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

set(VORBISIDEC_PUBLIC_HEADERS
    ivorbiscodec.h
    ivorbisfile.h
)

set(VORBISIDEC_HEADERS
    codebook.h
    misc.h
    mdct_lookup.h
    os.h
    mdct.h
    block.h
    lsp_lookup.h
    registry.h
    window.h
    window_lookup.h
    codec_internal.h
    backends.h
    asm_arm.h
)

set(VORBISIDEC_SOURCES
    mdct.c
    block.c
    window.c
    synthesis.c
    info.c
    floor1.c
    floor0.c
    vorbisfile.c
    res012.c
    mapping0.c
    registry.c
    codebook.c
    sharedbook.c
)

if(BUILD_FRAMEWORK)
    set(BUILD_SHARED_LIBS TRUE)
endif()

if(WIN32 AND BUILD_SHARED_LIBS)
    list(APPEND VORBISIDEC_SOURCES win32/vorbisidec.def)
endif()

add_library(vorbisidec ${VORBISIDEC_PUBLIC_HEADERS} ${VORBISIDEC_HEADERS} ${VORBISIDEC_SOURCES})
add_library(Tremor::vorbisidec ALIAS vorbisidec)
target_include_directories(vorbisidec PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_link_libraries(vorbisidec PUBLIC Ogg::ogg)

if(BUILD_LOW_ACCURACY)
    target_compile_definitions(vorbisidec PRIVATE -D_LOW_ACCURACY_)
endif()
if(HAVE_ALLOCA_H)
    target_compile_definitions(vorbisidec PRIVATE -DHAVE_ALLOCA_H)
endif()
if(HAVE_MEMORY_H)
    target_compile_definitions(vorbisidec PRIVATE -DHAVE_MEMORY_H)
endif()
if(HAVE_SYS_TYPES_H)
    target_compile_definitions(vorbisidec PRIVATE -DHAVE_SYS_TYPES_H)
endif()
if(HAVE_ALLOCA)
    target_compile_definitions(vorbisidec PRIVATE -DHAVE_ALLOCA)
endif()
if(HAVE_ARM_ASM)
    target_compile_definitions(vorbisidec PRIVATE -D_ARM_ASSEM_)
endif()
if(WORDS_BIGENDIAN)
    target_compile_definitions(vorbisidec PRIVATE -DWORDS_BIGENDIAN)
endif(WORDS_BIGENDIAN)

set_target_properties(
    vorbisidec PROPERTIES
    SOVERSION ${LIB_SOVERSION}
    VERSION ${LIB_VERSION}
    PUBLIC_HEADER "${VORBISIDEC_PUBLIC_HEADERS}"
)

if(BUILD_FRAMEWORK)
    set_target_properties(vorbisidec PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${PROJECT_VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER org.xiph.tremor
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${PROJECT_VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${PROJECT_VERSION}
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        OUTPUT_NAME Tremor
    )
endif()

configure_pkg_config_file(vorbisidec.pc.in)

install(TARGETS vorbisidec
    EXPORT TremorTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/tremor
)

export(EXPORT TremorTargets NAMESPACE Tremor:: FILE TremorTargets.cmake)

if(INSTALL_CMAKE_PACKAGE_MODULE)
    set(CMAKE_INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/Tremor)
    install(EXPORT TremorTargets
        DESTINATION ${CMAKE_INSTALL_CONFIGDIR}
        NAMESPACE Tremor::
    )

    include(CMakePackageConfigHelpers)

    configure_package_config_file(${PROJECT_SOURCE_DIR}/cmake/TremorConfig.cmake.in ${PROJECT_BINARY_DIR}/TremorConfig.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_CONFIGDIR}
        PATH_VARS CMAKE_INSTALL_FULL_INCLUDEDIR
    )

    write_basic_package_version_file(${PROJECT_BINARY_DIR}/TremorConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install(FILES ${PROJECT_BINARY_DIR}/TremorConfig.cmake ${PROJECT_BINARY_DIR}/TremorConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_CONFIGDIR}
    )
endif()

if(INSTALL_PKG_CONFIG_MODULE)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/vorbisidec.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()

set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
include(CPack)
