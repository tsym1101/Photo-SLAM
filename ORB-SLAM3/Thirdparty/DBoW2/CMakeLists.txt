cmake_minimum_required(VERSION 2.8)
project(DBoW2)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4 /O2")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -O3")
endif()

set(HDRS_DBOW2
  DBoW2/BowVector.h
  DBoW2/FORB.h 
  DBoW2/FClass.h       
  DBoW2/FeatureVector.h
  DBoW2/ScoringObject.h   
  DBoW2/TemplatedVocabulary.h)
set(SRCS_DBOW2
  DBoW2/BowVector.cpp
  DBoW2/FORB.cpp      
  DBoW2/FeatureVector.cpp
  DBoW2/ScoringObject.cpp)

set(HDRS_DUTILS
  DUtils/Random.h
  DUtils/Timestamp.h)
set(SRCS_DUTILS
  DUtils/Random.cpp
  DUtils/Timestamp.cpp)

find_package(OpenCV 4 QUIET)
if(NOT OpenCV_FOUND)
   find_package(OpenCV 3.0 QUIET)
   if(NOT OpenCV_FOUND)
      message(FATAL_ERROR "OpenCV > 3.0 not found.")
   endif()
endif()
find_package(Boost REQUIRED)

if(WIN32)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
  add_definitions(-DDBOW2_EXPORTS)
else()
  set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
endif()

include_directories(${OpenCV_INCLUDE_DIRS} ${Boost_INCLUDE_DIRS})

option(BUILD_SHARED_LIBS "Build DBoW2 as a shared library" ON)

if(BUILD_SHARED_LIBS)
  add_library(DBoW2 SHARED ${SRCS_DBOW2} ${SRCS_DUTILS})
  if(MSVC)
    set_target_properties(DBoW2 PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
  endif()
else()
  add_library(DBoW2 STATIC ${SRCS_DBOW2} ${SRCS_DUTILS})
endif()

target_link_libraries(DBoW2 ${OpenCV_LIBS})

install(TARGETS DBoW2
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)

install(FILES ${HDRS_DBOW2} DESTINATION include/DBoW2)
install(FILES ${HDRS_DUTILS} DESTINATION include/DUtils)