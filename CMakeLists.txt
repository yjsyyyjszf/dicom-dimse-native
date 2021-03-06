cmake_minimum_required(VERSION 3.9)
cmake_policy(SET CMP0042 NEW)
set (CMAKE_CXX_STANDARD 11)


#-------------------------------------DCMTK-----------------------------------

# As of 2018-12-26 DCMTK has been validated to build with CMake 3.13.2 policies.
set(DCMTK_MAX_CMAKE_POLICY_VERSION 3.13.2)

# Set and use the newest CMake policies that are validated to work
# (VERSION_LESS comparison is only defined for CMake 3 and newer)
if(CMAKE_MAJOR_VERSION LESS 3)
    set(DCMTK_CMAKE_POLICY_VERSION "${CMAKE_VERSION}")
elseif(CMAKE_VERSION VERSION_LESS DCMTK_MAX_CMAKE_POLICY_VERSION)
    set(DCMTK_CMAKE_POLICY_VERSION "${CMAKE_VERSION}")
else()
    set(DCMTK_CMAKE_POLICY_VERSION "${DCMTK_MAX_CMAKE_POLICY_VERSION}")
endif()
cmake_policy(VERSION "${DCMTK_CMAKE_POLICY_VERSION}")


# Declare project
project(DCMTK)

# Check the build system
include(CMake/dcmtkPrepare.cmake NO_POLICY_SCOPE)

#set(DCMTK_MODULES ofstd oflog dcmdata dcmimgle
#  dcmimage dcmjpeg dcmjpls dcmtls dcmnet
#  CACHE STRING "List of modules that should be built.")

set(DCMTK_MODULES ofstd oflog dcmdata dcmnet
  CACHE STRING "List of modules that should be built.")

#-----------------------------------------------------------------------------
# Include directories
#-----------------------------------------------------------------------------

set(DCMTK_INCLUDE_DIR "${DCMTK_BINARY_DIR}/config/include")
foreach(inc ${DCMTK_MODULES})
  list(APPEND DCMTK_INCLUDE_DIR "${DCMTK_SOURCE_DIR}/${inc}/include")
endforeach()

include_directories(${DCMTK_INCLUDE_DIR})

#-----------------------------------------------------------------------------
# Prepare osconfig.h
#-----------------------------------------------------------------------------

# add the osconfig.h.in file
configure_file("${DCMTK_SOURCE_DIR}/CMake/osconfig.h.in"
               "${DCMTK_BINARY_DIR}/config/include/dcmtk/config/osconfig.h")

#-----------------------------------------------------------------------------
# Prepare arith.h
#-----------------------------------------------------------------------------

INSPECT_FUNDAMENTAL_ARITHMETIC_TYPES()

# Add flags specific to the compiler
#-----------------------------------
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    message(STATUS "MSVC detected, adding compile flags")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W3")
else()
    message(STATUS "GNU or Clang detected, adding compile flags")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic -Wconversion -Wfloat-equal -fPIC")
    set (CMAKE_POSITION_INDEPENDENT_CODE TRUE)
endif()

#-----------------------------------------------------------------------------
# Start actual compilation tasks
#-----------------------------------------------------------------------------

# Recurse into subdirectories
foreach(module config ${DCMTK_MODULES})
  add_subdirectory(${module})
endforeach()
include(CMake/dcmtkAfterModules.cmake NO_POLICY_SCOPE)


# Build a shared library named after the project from the files in `src/`
file(GLOB SOURCE_FILES "src/*.cc" "src/*.h")
add_library(${PROJECT_NAME} SHARED 
	${SOURCE_FILES}
	)

# Define dependency libraries
#----------------------------
target_link_libraries(${PROJECT_NAME} dcmnet dcmdata oflog ofstd)

# Gives our library file a .node extension without any "lib" prefix
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")

# Include N-API wrappers
execute_process(COMMAND node -p "require('node-addon-api').include"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE NODE_ADDON_API_DIR
        )
string(REPLACE "\n" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REPLACE "\"" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})


# Essential include files to build a node addon,
# You should add this line in every CMake.js based project
target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_JS_INC} ${NODE_ADDON_API_DIR})

# Essential library files to link to a node addon
# You should add this line in every CMake.js based project
target_link_libraries(${PROJECT_NAME} ${CMAKE_JS_LIB})

# define NPI_VERSION
add_definitions(-DNAPI_VERSION=4)
