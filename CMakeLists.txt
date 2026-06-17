# Minimal CMake build for embedding msdfgen in the LuckyEngine CMake build.
# This branch's source is dev's premake-vendor lineage (no upstream CMakeLists);
# we add only what the engine needs: the msdfgen + msdfgen-ext static libs and
# their msdfgen::* aliases, consumed by msdf-atlas-gen's CMakeLists.
#
# Notes:
#  - System FreeType is used (find_package) to match the engine's freetype
#    handling; the bundled `freetype` submodule is left uninitialized.
#  - Skia is off, so ext/resolve-shape-geometry.cpp compiles to an empty TU.
#  - lodepng + tinyxml2 are amalgamated under lib/ (headers in include/).

cmake_minimum_required(VERSION 3.10)

project(msdfgen VERSION 1.9 LANGUAGES CXX)

option(MSDFGEN_USE_SKIA "Build with the Skia library" OFF)
option(MSDFGEN_USE_CPP11 "Build with C++11 enabled" ON)

if(NOT TARGET Freetype::Freetype)
    find_package(Freetype REQUIRED)
endif()

# ── Core library ────────────────────────────────────────────────────────────
file(GLOB_RECURSE msdfgen_SOURCES CONFIGURE_DEPENDS "core/*.cpp")
file(GLOB_RECURSE msdfgen_HEADERS CONFIGURE_DEPENDS "core/*.h" "core/*.hpp")
add_library(msdfgen STATIC ${msdfgen_SOURCES} ${msdfgen_HEADERS} msdfgen.h)
add_library(msdfgen::msdfgen ALIAS msdfgen)
target_include_directories(msdfgen PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

# ── Extensions library (font/svg/png import-export) ─────────────────────────
# lodepng + tinyxml2 are compiled in from lib/; their headers live in include/.
file(GLOB_RECURSE msdfgen-ext_SOURCES CONFIGURE_DEPENDS "ext/*.cpp" "lib/*.cpp")
file(GLOB_RECURSE msdfgen-ext_HEADERS CONFIGURE_DEPENDS "ext/*.h" "ext/*.hpp")
add_library(msdfgen-ext STATIC ${msdfgen-ext_SOURCES} ${msdfgen-ext_HEADERS} msdfgen-ext.h)
add_library(msdfgen::msdfgen-ext ALIAS msdfgen-ext)
target_include_directories(msdfgen-ext PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
target_link_libraries(msdfgen-ext PUBLIC msdfgen::msdfgen Freetype::Freetype)
target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_CMAKE_BUILD)

if(MSDFGEN_USE_CPP11)
    foreach(t msdfgen msdfgen-ext)
        target_compile_features(${t} PUBLIC cxx_std_11)
        target_compile_definitions(${t} PUBLIC MSDFGEN_USE_CPP11)
    endforeach()
endif()

set_target_properties(msdfgen msdfgen-ext PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    FOLDER "Dependencies"
)
