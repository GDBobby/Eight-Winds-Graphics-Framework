include(./.env.cmake OPTIONAL RESULT_VARIABLE LOCAL_ENV)
message(STATUS "Local .env.cmake: ${LOCAL_ENV}")

cmake_minimum_required(VERSION 3.23)

project(EightWindsGraphicsFramework VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(is_project_root OFF)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
	set(is_project_root ON)
endif()

option(EWE_EXAMPLES "Build examples" ${is_project_root})
option(EWE_PROJECTS "Build projects" ${is_project_root})

Message(STATUS "Cmake Generator : ${CMAKE_GENERATOR}")

file(GLOB_RECURSE SOURCES ${PROJECT_SOURCE_DIR}/src/*.cpp)
file(GLOB_RECURSE HEADER_FILES ${PROJECT_SOURCE_DIR}/include/*.h)
file(GLOB_RECURSE HEADER_FILES_PP ${PROJECT_SOURCE_DIR}/include/*.hpp)

include(FetchContent)

FetchContent_Declare(
  glfw
  GIT_REPOSITORY https://github.com/glfw/glfw
  GIT_TAG d79afa8
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/glfw"
  BINARY_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/glfw-build"
)

set(GLFW_BUILD_EXAMPLES OFF)
set(GLFW_BUILD_TESTS OFF)
set(GLFW_BUILD_DOCS OFF)
set(GLFW_INSTALL OFF)
set(GLFW_VULKAN_STATIC ON)

FetchContent_MakeAvailable(glfw)

#glslang
set(BUILD_SHARED_LIBS FALSE)
set(BUILD_EXTERNAL TRUE)
set(ENABLE_GLSLANG_BINARIES TRUE)
set(ENABLE_CTEST FALSE)
set(ENABLE_HLSL FALSE)
set(GLSLANG_TESTS FALSE)

add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/external/include/notvulkan/glslang")
include_directories("${CMAKE_CURRENT_SOURCE_DIR}/external/include/notvulkan/glslang")

set(GLSLANG_LIBS
	OSDependent
	SPIRV
	glslang
)

add_library(${PROJECT_NAME} STATIC ${SOURCES} ${HEADER_FILES} ${HEADER_FILES_PP})

# 1. Set VULKAN_SDK_PATH in .env.cmake to target specific vulkan version
if (DEFINED VULKAN_SDK_PATH)
  set(Vulkan_INCLUDE_DIR "${VULKAN_SDK_PATH}/Include") # 1.1 Make sure this include path is correct
  set(Vulkan_LIB "${VULKAN_SDK_PATH}/Lib") # 1.2 Make sure lib path is correct
  set(Vulkan_FOUND "True")
  if(UNIX)
  	set(Vulkan_LIBRARY libvulkan.so)
  else()
	set(Vulkan_LIBRARY vulkan-1)
  endif()
else()
  message(STATUS "Vulkan path undefined in .env")
  find_package(Vulkan REQUIRED) # throws error if could not find Vulkan
  message(STATUS "Found Vulkan package:  ${Vulkan_INCLUDE_DIRS}") #this line isnt working
endif()
if (NOT Vulkan_FOUND)
	message(FATAL_ERROR "Could not find Vulkan library!")
else()
	message(STATUS "Using vulkan lib at: ${Vulkan_LIBRARY}")
endif()

message(STATUS "including directories")
target_include_directories(${PROJECT_NAME} PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	$<INSTALL_INTERFACE:include>
	
	${Vulkan_INCLUDE_DIR}
	${CMAKE_CURRENT_SOURCE_DIR}/external/include
	
)
message(STATUS "linking directories")
target_link_directories(${PROJECT_NAME} PUBLIC
	${Vulkan_LIB}
)

message(STATUS "linking libs")


target_link_libraries(${PROJECT_NAME} 
	glfw
	${Vulkan_LIBRARY}
	${GLSLANG_LIBS}
)

set_target_properties(${PROJECT_NAME}
    PROPERTIES
    DEBUG_POSTFIX "-d"  # Appends "-d" in Debug mode
    RELEASE_POSTFIX ""   # No postfix in Release mode
)

# Create source groups to maintain file structure in Visual Studio
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${SOURCES} ${HEADER_FILES} ${HEADER_FILES_PP})
