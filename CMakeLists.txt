
cmake_minimum_required(VERSION 3.15)
include(cmake/version.cmake)

option(MSDF_ATLAS_INSTALL "Generate installation target" OFF)
option(MSDF_ATLAS_DYNAMIC_RUNTIME "Link dynamic runtime library instead of static" OFF)
option(BUILD_SHARED_LIBS "Generate dynamic library files instead of static" OFF)

if(NOT MSDF_ATLAS_MSDFGEN_EXTERNAL)
    set(MSDFGEN_CORE_ONLY OFF CACHE INTERNAL "Only build the core msdfgen library with no dependencies (disabled for msdf-atlas-gen)" FORCE)
    set(MSDFGEN_USE_CPP11 ON CACHE INTERNAL "Build with C++11 enabled (always enabled for msdf-atlas-gen)" FORCE)
    set(MSDFGEN_INSTALL ${MSDF_ATLAS_INSTALL} CACHE INTERNAL "Generate installation target for msdfgen" FORCE)
    set(MSDFGEN_DYNAMIC_RUNTIME ${MSDF_ATLAS_DYNAMIC_RUNTIME} CACHE INTERNAL "Link dynamic runtime library instead of static for msdfgen" FORCE)
endif()

get_property(MULTI_CONFIG GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT MULTI_CONFIG AND NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE not set, defaulting to Release")
    set(CMAKE_BUILD_TYPE Release)
endif()

if(MSDF_ATLAS_DYNAMIC_RUNTIME)
    set(MSDF_ATLAS_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
else()
    set(MSDF_ATLAS_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if(BUILD_SHARED_LIBS)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

# Version is specified in vcpkg.json
project(msdf-atlas-gen VERSION ${MSDF_ATLAS_VERSION} LANGUAGES CXX)


add_subdirectory(msdfgen)

file(GLOB_RECURSE MSDF_ATLAS_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "msdf-atlas-gen/*.h" "msdf-atlas-gen/*.hpp")
file(GLOB_RECURSE MSDF_ATLAS_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "msdf-atlas-gen/*.cpp")

# msdf-atlas-gen library
if (NOT DEFINED CMAKE_DEBUG_POSTFIX)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

add_library(msdf-atlas-gen ${MSDF_ATLAS_HEADERS} ${MSDF_ATLAS_SOURCES})
add_library(msdf-atlas-gen::msdf-atlas-gen ALIAS msdf-atlas-gen)
set_target_properties(msdf-atlas-gen PROPERTIES PUBLIC_HEADER "${MSDF_ATLAS_HEADERS}" DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")
set_property(TARGET msdf-atlas-gen PROPERTY MSVC_RUNTIME_LIBRARY "${MSDF_ATLAS_MSVC_RUNTIME}")
target_compile_definitions(msdf-atlas-gen PUBLIC
    MSDF_ATLAS_VERSION=${MSDF_ATLAS_VERSION}
    MSDF_ATLAS_VERSION_MAJOR=${MSDF_ATLAS_VERSION_MAJOR}
    MSDF_ATLAS_VERSION_MINOR=${MSDF_ATLAS_VERSION_MINOR}
    MSDF_ATLAS_VERSION_REVISION=${MSDF_ATLAS_VERSION_REVISION}
    MSDF_ATLAS_COPYRIGHT_YEAR=${MSDF_ATLAS_COPYRIGHT_YEAR}
)
target_include_directories(msdf-atlas-gen INTERFACE
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)
target_compile_definitions(msdf-atlas-gen PUBLIC MSDF_ATLAS_NO_ARTERY_FONT)
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT msdf-atlas-gen)

target_compile_features(msdf-atlas-gen PUBLIC cxx_std_11)
target_link_libraries(msdf-atlas-gen PUBLIC msdfgen::msdfgen)

if(BUILD_SHARED_LIBS AND WIN32)
    target_compile_definitions(msdf-atlas-gen PRIVATE "MSDF_ATLAS_PUBLIC=__declspec(dllexport)")
    target_compile_definitions(msdf-atlas-gen INTERFACE "MSDF_ATLAS_PUBLIC=__declspec(dllimport)")
else()
    target_compile_definitions(msdf-atlas-gen PUBLIC MSDF_ATLAS_PUBLIC=)
endif()

# Installation
if(MSDF_ATLAS_INSTALL)
    include(GNUInstallDirs)
    include(CMakePackageConfigHelpers)
    set(MSDF_ATLAS_CONFIG_PATH "lib/cmake/msdf-atlas-gen")

    # install tree package config
    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/msdf-atlas-gen-config-version.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    configure_package_config_file(
        cmake/msdf-atlas-gen-config.cmake.in
        ${MSDF_ATLAS_CONFIG_PATH}/msdf-atlas-gen-config.cmake
        INSTALL_DESTINATION ${MSDF_ATLAS_CONFIG_PATH}
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
    )

    # build tree package config
    configure_file(
        cmake/msdf-atlas-gen-config.cmake.in
        msdf-atlas-gen-config.cmake
        @ONLY
    )

    install(TARGETS msdf-atlas-gen EXPORT msdf-atlas-gen-targets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        FRAMEWORK DESTINATION ${CMAKE_INSTALL_LIBDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/msdf-atlas-gen
    )
    if(MSVC AND BUILD_SHARED_LIBS)
        install(FILES $<TARGET_PDB_FILE:msdf-atlas-gen> DESTINATION ${CMAKE_INSTALL_BINDIR} OPTIONAL)
    endif()

    export(EXPORT msdf-atlas-gen-targets NAMESPACE msdf-atlas-gen:: FILE "${CMAKE_CURRENT_BINARY_DIR}/msdf-atlas-gen-targets.cmake")
    install(EXPORT msdf-atlas-gen-targets FILE msdf-atlas-gen-targets.cmake NAMESPACE msdf-atlas-gen:: DESTINATION ${MSDF_ATLAS_CONFIG_PATH})

    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/${MSDF_ATLAS_CONFIG_PATH}/msdf-atlas-gen-config.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/msdf-atlas-gen-config-version.cmake"
        DESTINATION ${MSDF_ATLAS_CONFIG_PATH}
    )
endif()
