cmake_minimum_required(VERSION 3.12)

project(Lerc
        DESCRIPTION "Limited Error Raster Compression"
        HOMEPAGE_URL "https://github.com/Esri/lerc"
        VERSION 4.0.0) # Keep in sync with Lerc_c_api.h

include(GNUInstallDirs)

set(CMAKE_CXX_STANDARD 17)

file(GLOB SOURCES
    "src/LercLib/*"
    "src/LercLib/Lerc1Decode/*"
    "src/LercLib/include/*"
)

# Make an option, defaulting to shared libs, but allow -DBUILD_SHARED_LIBS=OFF
option (BUILD_SHARED_LIBS "Build shared libraries (set to OFF to build static libs)" ON)

# Include FetchContent module
include(FetchContent)

# Check for required system packages
find_package(X11 REQUIRED)
find_package(OpenGL REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(X11 REQUIRED x11 xrandr xi xcursor xinerama)

# Fetch OpenCV
FetchContent_Declare(
  opencv
  GIT_REPOSITORY https://github.com/opencv/opencv.git
  GIT_TAG 4.5.5  # Specify the version/tag/commit as needed
)

# Fetch raylib
FetchContent_Declare(
  raylib
  GIT_REPOSITORY https://github.com/raysan5/raylib.git
  GIT_TAG 4.0.0  # Specify the version/tag/commit as needed
)

# Fetch curl
FetchContent_Declare(
  curl_external
  GIT_REPOSITORY https://github.com/curl/curl.git
  GIT_TAG curl-7_76_1  # Specify the version/tag/commit as needed
)

# Make content available
FetchContent_MakeAvailable(opencv raylib)

# Find the curl library
find_package(CURL QUIET)
if(NOT CURL_FOUND)
  message(STATUS "curl not found, using FetchContent to download it.")
  FetchContent_MakeAvailable(curl_external)
  set(CURL_INCLUDE_DIRS ${curl_external_SOURCE_DIR}/include)
  set(CURL_LIBRARIES ${curl_external_BINARY_DIR}/libcurl.a)
endif()

# Add library
add_library(Lerc ${SOURCES})

set_target_properties(Lerc
    PROPERTIES
    PUBLIC_HEADER "src/LercLib/include/Lerc_types.h;src/LercLib/include/Lerc_c_api.h")

if(BUILD_SHARED_LIBS)
    set_target_properties(Lerc
        PROPERTIES
        SOVERSION ${CMAKE_PROJECT_VERSION_MAJOR}
        DEFINE_SYMBOL LERC_EXPORTS)
else()
    set_target_properties(Lerc
        PROPERTIES
        COMPILE_DEFINITIONS LERC_STATIC)
endif()

# Link the fetched libraries to Lerc
target_include_directories(Lerc PRIVATE ${CURL_INCLUDE_DIRS})
target_link_libraries(Lerc PRIVATE opencv_core opencv_imgcodecs opencv_highgui raylib ${CURL_LIBRARIES} ${X11_LIBRARIES} ${OPENGL_LIBRARIES})

install(
    TARGETS Lerc
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Handle both absolute paths (e.g. NixOS) and relative for a relocatable package
if(IS_ABSOLUTE("${CMAKE_INSTALL_INCLUDEDIR}"))
    set(LERC_PC_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}")
else()
    set(LERC_PC_INCLUDEDIR "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
endif()
if(IS_ABSOLUTE("${CMAKE_INSTALL_LIBDIR}"))
    set(LERC_PC_LIBDIR "${CMAKE_INSTALL_LIBDIR}")
else()
    set(LERC_PC_LIBDIR "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
endif()

# Configure and install pkgconfig file
configure_file(Lerc.pc.in ${CMAKE_CURRENT_BINARY_DIR}/Lerc.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/Lerc.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

message("Lerc built successfully with OpenCV, raylib, and curl!")
