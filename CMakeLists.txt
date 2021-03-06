cmake_minimum_required(VERSION 3.2.2)
project(pxp-agent)

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Defaulting to a release build.")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# define further options
option(EXTERNAL_CPP_PCP_CLIENT "ON - use an installed version of cpp-pcp-client. OFF - use the git submodule" ON)
option(TEST_VIRTUAL "ON - certain class member functions became virtual to enable mocking for unit tests" OFF)
option(DEV_LOG_COLOR "Enable colorization for logging (development setting)" OFF)

# Project Output Paths

set(MODULES_INSTALL_PATH pxp-agent/modules CACHE STRING  "Location to install core modulee. Can be an absolute path, or relative path from CMAKE_INSTALL_PREFIX.")

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
set(VENDOR_DIRECTORY ${PROJECT_SOURCE_DIR}/vendor)
list(APPEND CMAKE_MODULE_PATH ${VENDOR_DIRECTORY})
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

# set a version macro and expand related template
set(APPLICATION_VERSION_STRING "1.0.1")
configure_file(templates/version-inl.hpp ${CMAKE_BINARY_DIR}/generated/version-inl.hpp)

# set the root path macro and expand related template
set(ROOT_PATH ${PROJECT_SOURCE_DIR})
configure_file(templates/root_path.hpp ${CMAKE_BINARY_DIR}/generated/root_path.hpp)

# include cmake binaries for templates
include_directories(${CMAKE_BINARY_DIR}/generated)

# prefer openssl from ports
if ("${CMAKE_SYSTEM_NAME}" MATCHES "Darwin")
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} /opt/local/lib)
    set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} /opt/local/include)
endif()

# Set RPATH if not installing to a system library directory
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" INSTALL_IS_SYSTEM_DIR)
if ("${INSTALL_IS_SYSTEM_DIR}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()

# Leatherman it up
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/vendor/leatherman/cmake")
include(options)

# Set compiler flags
include(cflags) # Leatherman
set(PXP_AGENT_FLAGS "")
set(CMAKE_CXX_FLAGS "${LEATHERMAN_CXX_FLAGS} ${PXP_AGENT_FLAGS}")
add_definitions(${LEATHERMAN_DEFINITIONS})

set(LEATHERMAN_USE_LOCALE TRUE)
set(LEATHERMAN_USE_NOWIDE TRUE)
set(LEATHERMAN_USE_CATCH TRUE)
set(LEATHERMAN_USE_RAPIDJSON TRUE)
set(LEATHERMAN_USE_JSON_CONTAINER TRUE)
set(LEATHERMAN_USE_LOGGING TRUE)
set(LEATHERMAN_USE_FILE_UTIL TRUE)
set(LEATHERMAN_USE_EXECUTION TRUE)
set(LEATHERMAN_USE_UTIL TRUE)
set(LEATHERMAN_USE_WINDOWS TRUE)
leatherman_logging_line_numbers()
add_subdirectory("vendor/leatherman")

if(DEV_LOG_COLOR)
    add_definitions(-DDEV_LOG_COLOR)
endif()

# Find libraries
SET(CMAKE_FIND_LIBRARY_PREFIXES "" "lib")
SET(CMAKE_FIND_LIBRARY_SUFFIXES ".so" ".a" ".lib" ".dll")

find_package(Boost 1.54 REQUIRED
  COMPONENTS filesystem system date_time thread log regex random)

find_package(OpenSSL REQUIRED)

# Specify the .cmake files for vendored libraries
include(${VENDOR_DIRECTORY}/horsewhisperer.cmake)

if(EXTERNAL_CPP_PCP_CLIENT)
    find_package(cpp-pcp-client REQUIRED)
else()
    include(${VENDOR_DIRECTORY}/cpp-pcp-client.cmake)
endif()

# Add the main binary

add_subdirectory(lib)
add_subdirectory(exe)

# Add the module install (the exe install is handled in the exe subdir)
install(FILES modules/pxp-module-puppet
        DESTINATION ${MODULES_INSTALL_PATH}
        PERMISSIONS OWNER_WRITE OWNER_READ OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

# Add the test suite

if(TEST_VIRTUAL)
    add_definitions(-DTEST_VIRTUAL)
endif()

enable_testing()

add_test(
    NAME "pxp-agent\\ library\\ tests"
    COMMAND "${EXECUTABLE_OUTPUT_PATH}/pxp-agent-unittests"
)

# Add cpplint target
FILE (GLOB_RECURSE ALL_SOURCES lib/*.cc lib/*.hpp exe/*.cc exe/*.hpp)
add_cpplint_files(${ALL_SOURCES})
enable_cpplint()

add_cppcheck_dirs("${PROJECT_SOURCE_DIR}/lib" "${PROJECT_SOURCE_DIR}/exe")
enable_cppcheck()

