cmake_minimum_required(VERSION 2.8)

project( OpenARK )

set(CMAKE_CXX_STACK_SIZE "10000000")
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake_modules")
option(BUILD_TESTS "BUILD_TESTS" OFF)
option(USE_PMD "USE_PMD" OFF) 

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
    message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

set(Boost_USE_STATIC_LIBS ON) 
set(Boost_USE_STATIC ON)

find_package( PCL REQUIRED )

find_package( OpenCV REQUIRED )
if(OpenCV_FOUND)
   message(STATUS "Found OpenCV: ${OpenCV_INCLUDE_DIRS}")
endif(OpenCV_FOUND)

if(NOT USE_PMD)
  find_package(RSSDK)
  if(RSSDK_FOUND)
       message(STATUS "Found RSSDK: ${RSSDK_INCLUDE_DIR}")
  endif(RSSDK_FOUND)
endif(NOT USE_PMD)

if(USE_PMD OR NOT RSSDK_FOUND)
  find_package( PMDSDK REQUIRED)
  if(PMDSDK_FOUND)
     message(STATUS "Found PMDSDK: ${PMDSDK_INCLUDE_DIR}")
  endif(PMDSDK_FOUND)
endif(USE_PMD OR NOT RSSDK_FOUND)

include_directories(
  ${OpenCV_INCLUDE_DIRS}
  ${PCL_INCLUDE_DIRS}
)

set(
  DEPENDENCIES  
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
)

add_definitions(
  ${PCL_DEFINITIONS}
)
foreach(DEFINITION ${PCL_DEFINITIONS})
  set(TARGET_COMPILE_FLAGS "${TARGET_COMPILE_FLAGS} ${DEFINITION}")
endforeach()

set(
  SOURCES 
  Object3D.cpp 
  Hand.cpp Plane.cpp 
  Webcam.cpp 
  DepthCamera.cpp 
  RGBCamera.cpp
  UDPSender.cpp 
  StreamingAverager.cpp 
  Calibration.cpp 
  Util.cpp	
  Visualizer.cpp	
  Converter.cpp
  HandFeatureExtractor.cpp
  HandClassifier.cpp
  libSVM/svm.cpp
)

if(NOT RSSDK_FOUND)
  SET(RSSDK "//")
else()
  set(SOURCES ${SOURCES} SR300Camera.cpp)
endif(NOT RSSDK_FOUND)

if(NOT PMDSDK_FOUND)
  SET(PMDSDK "//")
else()
  include_directories(${PMDSDK_INCLUDE_DIR})
  set(SOURCES ${SOURCES} PMDCamera.cpp)
  set(DEPENDENCIES ${DEPENDENCIES} ${PMDSDK_LIBRARIES})
endif(NOT PMDSDK_FOUND)

configure_file(Version.h.in ${PROJECT_SOURCE_DIR}/version.h)

macro(ADD_MSVC_PRECOMPILED_HEADER PrecompiledHeader PrecompiledSource SourcesVar)
  if(MSVC)
    get_filename_component(PrecompiledBasename ${PrecompiledHeader} NAME_WE)
    set(PrecompiledBinary "$(IntDir)/${PrecompiledBasename}.pch")
    set(Sources ${${SourcesVar}})

    if(NOT ${PrecompiledSource} STREQUAL "")
        set(Sources ${Sources} main.cpp)
        set_source_files_properties(${PrecompiledSource}
                                    PROPERTIES COMPILE_FLAGS "/Yc\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                               OBJECT_OUTPUTS "${PrecompiledBinary}")
    endif(NOT ${PrecompiledSource} STREQUAL "")

    set_source_files_properties(${Sources}
                                PROPERTIES COMPILE_FLAGS "/Yu\"${PrecompiledHeader}\" /FI\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                           OBJECT_DEPENDS "${PrecompiledBinary}")  
    list(APPEND ${SourcesVar} ${PrecompiledSource})
  endif(MSVC)
endmacro(ADD_MSVC_PRECOMPILED_HEADER)

if (MSVC)
    ADD_MSVC_PRECOMPILED_HEADER("stdafx.h" "stdafx.cpp" SOURCES)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif(MSVC)

add_executable( OpenARK main.cpp ${SOURCES} )
target_link_libraries( OpenARK ${DEPENDENCIES} )
set_target_properties( OpenARK PROPERTIES
  COMPILE_FLAGS ${TARGET_COMPILE_FLAGS})

if(${BUILD_TESTS})
  if(NOT EXISTS ${PROJECT_SOURCE_DIR}/OpenARK_test)
    execute_process(
      COMMAND git clone https://github.com/augcog/OpenARK_test.git
      WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})
  #else()
  #  execute_process(
  #  COMMAND git pull
  #    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/OpenARK_test)
  endif(NOT EXISTS ${PROJECT_SOURCE_DIR}/OpenARK_test)

  set(
    TEST_SOURCES 
    OpenARK_test/test.cpp
    OpenARK_test/TestCamera.cpp
  )

  ADD_MSVC_PRECOMPILED_HEADER("../stdafx.h" "" TEST_SOURCES)

  add_executable( OpenARK_test OpenARK_test/test.cpp 
    OpenARK_test/TestCamera.cpp ${SOURCES})
  target_link_libraries( OpenARK_test ${DEPENDENCIES} )
  set_target_properties( OpenARK_test PROPERTIES
    COMPILE_FLAGS ${TARGET_COMPILE_FLAGS})
endif(${BUILD_TESTS})



