cmake_minimum_required(VERSION 3.16)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

# See docs/release_checklist.md
set(MAJOR_VERSION 3)
set(MINOR_VERSION 0)
set(MICRO_VERSION 0)
set(SDL_REQUIRED_VERSION 3.0.0)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
    message(FATAL_ERROR "Prevented in-tree built. Please create a build directory outside of the SDL_ttf source code and call cmake from there")
endif()

project(SDL3_ttf
    LANGUAGES C
    VERSION "${MAJOR_VERSION}.${MINOR_VERSION}.${MICRO_VERSION}"
)

include(PrivateSdlFunctions)
sdl_calculate_derived_version_variables(${MAJOR_VERSION} ${MINOR_VERSION} ${MICRO_VERSION})

message(STATUS "Configuring ${PROJECT_NAME} ${PROJECT_VERSION}")

if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)
    set(SDL3TTF_ROOTPROJECT ON)
else()
    set(SDL3TTF_ROOTPROJECT OFF)
endif()

# Assume MSVC projects don't have a package manager and need vendored dependencies (by default).
# Most other platforms have some kind of package manager.
# FIXME: consider a package manager such as conan/vcpkg instead of vendoring
if(ANDROID OR MSVC)
    set(vendored_default TRUE)
else()
    set(vendored_default FALSE)
endif()

include(CMakeDependentOption)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)
include(CheckSymbolExists)

option(CMAKE_POSITION_INDEPENDENT_CODE "Build static libraries with -fPIC" ON)
option(BUILD_SHARED_LIBS "Build the library as a shared library" ON)

option(SDL3TTF_INSTALL "Enable SDL3_ttf install target" ${SDL3TTF_ROOTPROJECT})
option(SDL3TTF_VENDORED "Use vendored third-party libraries" ${vendored_default})
option(SDL3TTF_WERROR "Treat warnings as errors" OFF)

option(SDL3TTF_SAMPLES "Build the SDL3_ttf sample program(s)" ${SDL3TTF_ROOTPROJECT})
cmake_dependent_option(SDL3TTF_SAMPLES_INSTALL "Install the SDL3_ttf sample program(s)" OFF "SDL3TTF_SAMPLES;SDL3TTF_INSTALL" OFF)

# For style consistency, create a SDL3TTF_FREETYPE CMake variable. This variable is NOT configurable.
set(SDL3TTF_FREETYPE ON)
set(SDL3TTF_FREETYPE_VENDORED "${SDL3TTF_VENDORED}")

option(SDL3TTF_HARFBUZZ "Use harfbuzz to improve text shaping" OFF)
set(SDL3TTF_HARFBUZZ_VENDORED "${SDL3TTF_VENDORED}")

# Save BUILD_SHARED_LIBS variable
set(SDL3TTF_BUILD_SHARED_LIBS "${BUILD_SHARED_LIBS}")

if(SDL3TTF_BUILD_SHARED_LIBS)
    set(sdl3_ttf_export_name SDL3_ttf)
    set(sdl3_ttf_install_name_infix shared)
    set(sdl3_target_name SDL3::SDL3)
else()
    set(sdl3_ttf_export_name SDL3_ttf-static)
    set(sdl3_ttf_install_name_infix static)
    set(sdl3_target_name SDL3::SDL3-static)
endif()

if(NOT TARGET ${sdl3_target_name})
    find_package(SDL3 ${SDL_REQUIRED_VERSION} REQUIRED)
endif()

add_library(sdl3ttf_build_options INTERFACE)
if(MSVC)
    target_compile_options(sdl3ttf_build_options INTERFACE /W2)
else()
    target_compile_options(sdl3ttf_build_options INTERFACE -Wall -Wextra)
endif()
if(SDL3TTF_WERROR)
    if(MSVC)
        target_compile_options(sdl3ttf_build_options INTERFACE /WX)
    else()
        target_compile_options(sdl3ttf_build_options INTERFACE -Werror)
    endif()
endif()

# Enable large file support on 32-bit glibc, so that the vendored libraries
# can access files with large inode numbers
check_symbol_exists("__GLIBC__" "stdlib.h" LIBC_IS_GLIBC)
if (LIBC_IS_GLIBC AND CMAKE_SIZEOF_VOID_P EQUAL 4)
    add_compile_definitions(_FILE_OFFSET_BITS=64)
endif()

add_library(SDL3_ttf
    SDL_ttf.c
)
add_library(SDL3_ttf::${sdl3_ttf_export_name} ALIAS SDL3_ttf)
target_include_directories(SDL3_ttf PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>/include"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>/include/SDL3"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL3>"
)
target_compile_definitions(SDL3_ttf PRIVATE
    BUILD_SDL
    SDL_BUILD_MAJOR_VERSION=${MAJOR_VERSION}
    SDL_BUILD_MINOR_VERSION=${MINOR_VERSION}
    SDL_BUILD_MICRO_VERSION=${MICRO_VERSION}
)
target_link_libraries(SDL3_ttf PRIVATE $<BUILD_INTERFACE:${sdl3_target_name}>)
target_link_libraries(SDL3_ttf PRIVATE $<BUILD_INTERFACE:sdl3ttf_build_options>)
if(WIN32 AND SDL3TTF_BUILD_SHARED_LIBS)
    target_sources(SDL3_ttf PRIVATE
        version.rc
    )
endif()
set_target_properties(SDL3_ttf PROPERTIES
    DEFINE_SYMBOL DLL_EXPORT
    EXPORT_NAME ${sdl3_ttf_export_name}
    C_VISIBILITY_PRESET "hidden"
)
if(NOT ANDROID)
    if(APPLE)
        # the SOVERSION property corresponds to the compatibility version and VERSION corresponds to the current version
        # https://cmake.org/cmake/help/latest/prop_tgt/SOVERSION.html#mach-o-versions
        set_target_properties(SDL3_ttf PROPERTIES
            SOVERSION "${DYLIB_COMPAT_VERSION}"
            VERSION "${DYLIB_CURRENT_VERSION}"
        )
    else()
        set_target_properties(SDL3_ttf PROPERTIES
            SOVERSION "${SO_VERSION_MAJOR}"
            VERSION "${SO_VERSION}"
        )
    endif()
endif()
if(SDL3TTF_BUILD_SHARED_LIBS)
    if(WIN32)
        set_target_properties(SDL3_ttf PROPERTIES
            PREFIX ""
        )
    endif()
else()
    if(MSVC)
        set_target_properties(SDL3_ttf PROPERTIES
            OUTPUT_NAME "SDL3_ttf-static"
        )
    endif()
endif()

if(SDL3TTF_BUILD_SHARED_LIBS)
    # Use `Compatible Interface Properties` to ensure a shared SDL3_ttf is linked to a shared SDL3 library
    set_property(TARGET SDL3_ttf PROPERTY INTERFACE_SDL3_SHARED ${SDL3TTF_BUILD_SHARED_LIBS})
    set_property(TARGET SDL3_ttf APPEND PROPERTY COMPATIBLE_INTERFACE_BOOL SDL3_SHARED)
endif()

set(INSTALL_EXTRA_TARGETS)
set(PC_LIBS)
set(PC_REQUIRES)

# Build freetype and harfbuzz as a static library
set(BUILD_SHARED_LIBS OFF)

if(SDL3TTF_HARFBUZZ)
    if(SDL3TTF_HARFBUZZ_VENDORED)
        message(STATUS "${PROJECT_NAME}: Using vendored harfbuzz library")
        # HB_BUILD_UTILS variable is used by harfbuzz
        set(HB_BUILD_UTILS OFF CACHE BOOL "harfbuzz build utils" FORCE)
        # SKIP_INSTALL_LIBRARIES variable is used by harfbuzz
        set(SKIP_INSTALL_LIBRARIES ON CACHE BOOL "harfbuzz install option" FORCE)
        # HB_HAVE_FREETYPE variable is used by harfbuzz
        set(HB_HAVE_FREETYPE ${SDL3TTF_FREETYPE} CACHE BOOL "harfbuzz freetype helpers" FORCE)
        if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/harfbuzz/CMakeLists.txt")
            message(FATAL_ERROR "No harfbuzz sources found. Install a harfbuzz development package or run the download script in the external folder.")
        endif()
        add_subdirectory(external/harfbuzz EXCLUDE_FROM_ALL)
        # harfbuzz is a c++ project, enable c++ here to ensure linking to the c++ standard library
        enable_language(CXX)
        if(NOT SDL3TTF_BUILD_SHARED_LIBS)
            list(APPEND INSTALL_EXTRA_TARGETS harfbuzz)
            list(APPEND PC_LIBS -l$<TARGET_FILE_BASE_NAME:harfbuzz>)
        endif()
        if(NOT TARGET harfbuzz::harfbuzz)
            add_library(harfbuzz::harfbuzz ALIAS harfbuzz)
        endif()
    else()
        message(STATUS "${PROJECT_NAME}: Using system harfbuzz library")
        find_package(harfbuzz REQUIRED)
        list(APPEND PC_REQUIRES harfbuzz)
    endif()
    target_compile_definitions(SDL3_ttf PRIVATE TTF_USE_HARFBUZZ=1)
    target_link_libraries(SDL3_ttf PRIVATE harfbuzz::harfbuzz)
endif()

if(SDL3TTF_FREETYPE)
    if(SDL3TTF_FREETYPE_VENDORED)
        message(STATUS "${PROJECT_NAME}: Using vendored freetype library")
        # FT_DISABLE_ZLIB variable is used by freetype
        set(FT_DISABLE_ZLIB ON CACHE BOOL "freetype zlib option")
        # FT_DISABLE_BZIP2 variable is used by freetype
        set(FT_DISABLE_BZIP2 ON CACHE BOOL "freetype bzip2 option")
        # FT_DISABLE_PNG variable is used by freetype
        set(FT_DISABLE_PNG ON CACHE BOOL "freetype png option")
        # FT_DISABLE_BROTLI variable is used by freetype
        set(FT_DISABLE_BROTLI ON CACHE BOOL "freetype option")
        if(SDL3TTF_HARFBUZZ)
            # FT_DISABLE_HARFBUZZ variable is used by freetype
            set(FT_DISABLE_HARFBUZZ OFF CACHE BOOL "freetype harfbuzz option" FORCE)
            # FT_REQUIRE_HARFBUZZ variable is used by freetype
            set(FT_REQUIRE_HARFBUZZ ON CACHE BOOL "freetype harfbuzz option" FORCE)
        else()
            # FT_DISABLE_HARFBUZZ variable is used by freetype
            set(FT_DISABLE_HARFBUZZ ON CACHE BOOL "freetype harfbuzz option" FORCE)
            # FT_REQUIRE_HARFBUZZ variable is used by freetype
            set(FT_REQUIRE_HARFBUZZ OFF CACHE BOOL "freetype harfbuzz option" FORCE)
        endif()
        if(NOT EXISTS "${PROJECT_SOURCE_DIR}/external/freetype/CMakeLists.txt")
            message(FATAL_ERROR "No freetype sources found. Install a freetype development package or run the download script in the external folder.")
        endif()
        add_subdirectory(external/freetype EXCLUDE_FROM_ALL)
        if(NOT TARGET Freetype::Freetype)
            add_library(Freetype::Freetype ALIAS freetype)
        endif()
        if(NOT SDL3TTF_BUILD_SHARED_LIBS)
            list(APPEND INSTALL_EXTRA_TARGETS freetype)
            list(APPEND PC_LIBS -l$<TARGET_FILE_BASE_NAME:freetype>)
        endif()
    else()
        message(STATUS "${PROJECT_NAME}: Using system freetype library")
        find_package(Freetype REQUIRED)
        list(APPEND PC_REQUIRES freetype2)
    endif()
    target_link_libraries(SDL3_ttf PRIVATE Freetype::Freetype)
endif()

# Restore BUILD_SHARED_LIBS variable
set(BUILD_SHARED_LIBS ${SDL3TTF_BUILD_SHARED_LIBS})

if(SDL3TTF_INSTALL)
    install(
        TARGETS SDL3_ttf
        EXPORT SDL3_ttfTargets
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT library
    )
    install(FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/include/SDL3/SDL_ttf.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL3" COMPONENT devel
    )

    if(INSTALL_EXTRA_TARGETS)
        install(TARGETS ${INSTALL_EXTRA_TARGETS}
            EXPORT SDL3_ttfTargets
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
            LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
            RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT library
            PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}" COMPONENT devel
        )
    endif()

    if(WIN32 AND NOT MINGW)
        set(SDL3TTF_INSTALL_CMAKEDIR_DEFAULT "cmake")
    else()
        set(SDL3TTF_INSTALL_CMAKEDIR_DEFAULT "${CMAKE_INSTALL_LIBDIR}/cmake/SDL3_ttf")
    endif()
    set(SDL3TTF_INSTALL_CMAKEDIR_ROOT "${SDL3TTF_INSTALL_CMAKEDIR_DEFAULT}" CACHE STRING "Location where to install SDL3_ttfConfig.cmake")

    if(WIN32 AND NOT MINGW)
        set(SDL3TTF_INSTALL_CMAKEDIR "${SDL3TTF_INSTALL_CMAKEDIR_ROOT}")
        set(LICENSES_PREFIX "licenses/SDL3_ttf")
    else()
        set(SDL3TTF_INSTALL_CMAKEDIR "${SDL3TTF_INSTALL_CMAKEDIR_ROOT}/SDL3_ttf")
        set(LICENSES_PREFIX "${CMAKE_INSTALL_DATAROOTDIR}/licenses/SDL3_ttf")
    endif()

    configure_package_config_file(cmake/SDL3_ttfConfig.cmake.in SDL3_ttfConfig.cmake
        INSTALL_DESTINATION "${SDL3TTF_INSTALL_CMAKEDIR}"
    )
    write_basic_package_version_file("${PROJECT_BINARY_DIR}/SDL3_ttfConfigVersion.cmake"
        COMPATIBILITY AnyNewerVersion
    )
    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/SDL3_ttfConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/SDL3_ttfConfigVersion.cmake"
        DESTINATION ${SDL3TTF_INSTALL_CMAKEDIR}
        COMPONENT devel
    )
    install(EXPORT SDL3_ttfTargets
        FILE SDL3_ttf-${sdl3_ttf_install_name_infix}-targets.cmake
        NAMESPACE SDL3_ttf::
        DESTINATION "${SDL3TTF_INSTALL_CMAKEDIR}"
        COMPONENT devel
    )

    string(JOIN " " PC_REQUIRES ${PC_REQUIRES})
    string(JOIN " " PC_LIBS ${PC_LIBS})
    configure_file(cmake/sdl3-ttf.pc.in sdl3-ttf.pc @ONLY)

    if(CMAKE_SYSTEM_NAME MATCHES FreeBSD)
        # FreeBSD uses ${PREFIX}/libdata/pkgconfig
        set(PC_DESTDIR "libdata/pkgconfig")
    else()
        set(PC_DESTDIR "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
    endif()
    # Always install sdl3-ttf.pc file: libraries might be different between config modes
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sdl3-ttf.pc"
        DESTINATION "${PC_DESTDIR}" COMPONENT devel)

    install(FILES "LICENSE.txt"
        DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/licenses/${PROJECT_NAME}"
        COMPONENT library
    )
endif()

if(SDL3TTF_SAMPLES)
    add_executable(glfont glfont.c)
    add_executable(showfont showfont.c)

    set(OpenGL_GL_PREFERENCE GLVND)
    find_package(OpenGL)
    if(TARGET OpenGL::OpenGL)
        target_compile_definitions(glfont PRIVATE HAVE_OPENGL)
        target_link_libraries(glfont PRIVATE OpenGL::OpenGL)
    elseif(TARGET OpenGL::GL)
        target_compile_definitions(glfont PRIVATE HAVE_OPENGL)
        target_link_libraries(glfont PRIVATE OpenGL::GL)
    endif()

    foreach(prog glfont showfont)
        target_link_libraries(${prog} PRIVATE SDL3_ttf::${sdl3_ttf_export_name})
        target_link_libraries(${prog} PRIVATE ${sdl3_target_name})

        if(SDL3TTF_SAMPLES_INSTALL)
            install(TARGETS ${prog}
                RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
                )
        endif()
    endforeach()
endif()
