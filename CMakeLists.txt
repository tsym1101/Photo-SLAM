cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(photo-slam LANGUAGES CXX CUDA)

if(NOT CMAKE_BUILD_TYPE)
    # set(CMAKE_BUILD_TYPE Debug)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CUDA_STANDARD 17)

set(CAFFE2_USE_CUDNN on)

# Uncomment the following line if you want to solve CMake Warning "Failed to compute shorthash for libnvrtc.so"
# set(PYTHON_EXECUTABLE "/usr/bin/python3")

find_package(Torch REQUIRED)

if(WIN32)
    # Remove pthread flag on Windows
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-deprecated-declarations")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS} -pthread")
endif()

# set(ENV{CUDA_LAUNCH_BLOCKING} 1)
# set(ENV{TORCH_USE_CUDA_DSA} 1)

find_package(CUDA REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(OpenCV 4 REQUIRED)
find_package(jsoncpp REQUIRED)

find_package(OpenGL REQUIRED)
find_package(glm REQUIRED)
find_package(glfw3 REQUIRED)
find_package(GLEW REQUIRED)

find_package(realsense2)
find_package(Boost REQUIRED COMPONENTS serialization)

set(ORB_SLAM3_SOURCE_DIR "${PROJECT_SOURCE_DIR}/ORB-SLAM3")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)

include_directories(
    ${PROJECT_SOURCE_DIR}
    ${ORB_SLAM3_SOURCE_DIR}
    ${ORB_SLAM3_SOURCE_DIR}/include
    ${ORB_SLAM3_SOURCE_DIR}/include/CameraModels
    ${ORB_SLAM3_SOURCE_DIR}/Thirdparty/Sophus
    ${EIGEN3_INCLUDE_DIR}
    ${OPENGL_INCLUDE_DIR})

##################################################################################
##  Build the gaussian splatting component libraries to ${PROJECT_SOURCE_DIR}/lib
##################################################################################

add_library(simple_knn SHARED
    third_party/simple-knn/simple_knn.cu
    third_party/simple-knn/simple_knn.h
    third_party/simple-knn/spatial.cu
    third_party/simple-knn/spatial.h)
# target_compile_features(simple_knn PUBLIC cxx_std_17)
target_link_libraries(simple_knn ${TORCH_LIBRARIES})

add_library(cuda_rasterizer SHARED
    include/operate_points.h
    src/operate_points.cu
    include/rasterize_points.h
    src/rasterize_points.cu
    include/stereo_vision.h
    src/stereo_vision.cu
    cuda_rasterizer/auxiliary.h
    cuda_rasterizer/backward.cu
    cuda_rasterizer/backward.h
    cuda_rasterizer/config.h
    cuda_rasterizer/forward.cu
    cuda_rasterizer/forward.h
    cuda_rasterizer/operate_points.h
    cuda_rasterizer/rasterizer.h
    cuda_rasterizer/rasterizer_impl.cu
    cuda_rasterizer/rasterizer_impl.h
    cuda_rasterizer/stereo_vision.h)
set_target_properties(cuda_rasterizer PROPERTIES CUDA_ARCHITECTURES "75;86")
# target_compile_features(cuda_rasterizer PUBLIC cxx_std_17)
target_include_directories(cuda_rasterizer PRIVATE
    ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES})
target_link_libraries(cuda_rasterizer
    glm::glm
    ${TORCH_LIBRARIES}
    Eigen3::Eigen)

##################################################################################
##  Build the ImGui library to ${PROJECT_SOURCE_DIR}/lib
##################################################################################

add_library(imgui SHARED
    viewer/imgui/imconfig.h
    viewer/imgui/imgui_demo.cpp
    viewer/imgui/imgui_draw.cpp
    viewer/imgui/imgui_impl_glfw.cpp
    viewer/imgui/imgui_impl_glfw.h
    viewer/imgui/imgui_impl_opengl3_loader.h
    viewer/imgui/imgui_impl_opengl3.cpp
    viewer/imgui/imgui_impl_opengl3.h
    viewer/imgui/imgui_internal.h
    viewer/imgui/imgui_tables.cpp
    viewer/imgui/imgui_widgets.cpp
    viewer/imgui/imgui.cpp
    viewer/imgui/imgui.h
    viewer/imgui/imstb_rectpack.h
    viewer/imgui/imstb_textedit.h
    viewer/imgui/imstb_truetype.h)
target_link_libraries(imgui glfw OpenGL::GL)

##################################################################################
##  Build the gaussian mapper library to ${PROJECT_SOURCE_DIR}/lib
##################################################################################

add_library(gaussian_mapper SHARED
    third_party/tinyply/tinyply.h
    third_party/tinyply/tinyply.cpp
    include/gaussian_keyframe.h
    include/gaussian_model.h
    include/gaussian_parameters.h
    include/gaussian_rasterizer.h
    include/gaussian_renderer.h
    include/gaussian_scene.h
    include/gaussian_trainer.h
    include/gaussian_mapper.h
    include/general_utils.h
    include/graphics_utils.h
    include/loss_utils.h
    include/sh_utils.h
    include/tensor_utils.h
    include/camera.h
    include/point_cloud.h
    include/point2d.h
    include/point3d.h
    include/types.h
    src/gaussian_keyframe.cpp
    src/gaussian_model.cpp
    src/gaussian_parameters.cpp
    src/gaussian_rasterizer.cpp
    src/gaussian_renderer.cpp
    src/gaussian_scene.cpp
    src/gaussian_trainer.cpp
    src/gaussian_mapper.cpp)

# Set library extension dynamically
if(WIN32)
    set(LIB_EXTENSION ".dll")
    # add_definitions(-DBYTE_DEFINED) # Disable byte definition from Windows SDK
    add_definitions(-DNOMINMAX -DWIN32_LEAN_AND_MEAN -D_HAS_STD_BYTE=0)
    target_include_directories(gaussian_mapper PUBLIC
        ${EIGEN3_INCLUDE_DIR}
        ${Boost_INCLUDE_DIRS})
else()
    set(LIB_EXTENSION ".so")
endif()
target_link_libraries(gaussian_mapper
    #${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3${LIB_EXTENSION}
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${OpenCV_LIBRARIES}
    jsoncpp_lib
    ${TORCH_LIBRARIES}
    Eigen3::Eigen
    # simple_knn
    # cuda_rasterizer
    ${PROJECT_SOURCE_DIR}/_build/Release/simple_knn.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/cuda_rasterizer.lib
    ${Boost_LIBRARIES})
add_dependencies(gaussian_mapper simple_knn cuda_rasterizer)

##################################################################################
##  Build the viewer library to ${PROJECT_SOURCE_DIR}/lib
##################################################################################

add_library(gaussian_viewer SHARED
    viewer/drawer_utils.h
    viewer/imgui_viewer.cpp
    viewer/imgui_viewer.h
    viewer/map_drawer.cpp
    viewer/map_drawer.h)
target_include_directories(gaussian_viewer PUBLIC
    ${GLEW_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS})
target_link_libraries(gaussian_viewer
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # imgui
    ${PROJECT_SOURCE_DIR}/_build/Release/imgui.lib
    ${OpenCV_LIBRARIES}
    jsoncpp_lib
    ${TORCH_LIBRARIES}
    glm::glm
    glfw
    OpenGL::GL
    ${Boost_LIBRARIES})
add_dependencies(gaussian_viewer gaussian_mapper imgui)

##################################################################################
##  Build the test examples to ${PROJECT_SOURCE_DIR}/bin
##################################################################################

# This is a C++ libtorch implementation of gaussian-splatting (https://github.com/graphdeco-inria/gaussian-splatting)
add_executable(train_colmap examples/train_colmap.cpp)
target_include_directories(train_colmap PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(train_colmap
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(train_colmap gaussian_mapper gaussian_viewer)
add_executable(view_result examples/view_result.cpp)
target_include_directories(view_result PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(view_result
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(view_result gaussian_mapper gaussian_viewer)

##################################################################################
##  Build the mapping examples to ${PROJECT_SOURCE_DIR}/bin
##################################################################################

# Replica Monocular
add_executable(replica_mono examples/replica_mono.cpp)
target_include_directories(replica_mono PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(replica_mono
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(replica_mono gaussian_mapper gaussian_viewer)

# Replica Monocular
add_executable(replica_rgbd examples/replica_rgbd.cpp)
target_include_directories(replica_rgbd PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(replica_rgbd
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(replica_rgbd gaussian_mapper gaussian_viewer)

# TUM Monocular
add_executable(tum_mono examples/tum_mono.cpp)
target_include_directories(tum_mono PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(tum_mono
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(tum_mono gaussian_mapper gaussian_viewer)

# TUM RGBD
add_executable(tum_rgbd examples/tum_rgbd.cpp)
target_include_directories(tum_rgbd PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(tum_rgbd
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(tum_rgbd gaussian_mapper gaussian_viewer)

# EuRoC Stereo
add_executable(euroc_stereo examples/euroc_stereo.cpp)
target_include_directories(euroc_stereo PUBLIC
    ${TORCH_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
    ${jsoncpp_INCLUDE_DIRS}
    ${GLEW_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
    ${GLFW_INCLUDE_DIRS})
target_link_libraries(euroc_stereo
    # gaussian_viewer
    # gaussian_mapper
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
    ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
    # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
    ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
    ${TORCH_LIBRARIES}
    ${Boost_LIBRARIES}
    jsoncpp_lib
    glm::glm
    ${OpenCV_LIBS}
    glfw)
add_dependencies(euroc_stereo gaussian_mapper gaussian_viewer)

##################################################################################
##  Build the mapping examples to ${PROJECT_SOURCE_DIR}/bin
##################################################################################

# Intel Realsense
# if(realsense2_FOUND)
    add_executable(realsense_rgbd examples/realsense_rgbd.cpp)
    target_include_directories(realsense_rgbd PUBLIC
        ${realsense_INCLUDE_DIR}
        ${TORCH_INCLUDE_DIRS}
        ${OpenCV_INCLUDE_DIRS}
        ${Boost_INCLUDE_DIRS}
        ${jsoncpp_INCLUDE_DIRS}
        ${GLEW_INCLUDE_DIRS}
        ${GLM_INCLUDE_DIRS})
    target_link_libraries(realsense_rgbd
        # gaussian_viewer
        # gaussian_mapper
        ${TORCH_LIBRARIES}
        ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_viewer.lib
        ${PROJECT_SOURCE_DIR}/_build/Release/gaussian_mapper.lib
        # ${ORB_SLAM3_SOURCE_DIR}/lib/libORB_SLAM3.so
        ${ORB_SLAM3_SOURCE_DIR}/lib/ORB_SLAM3.lib
        ${realsense2_LIBRARY}
        ${Boost_LIBRARIES}
        jsoncpp_lib
        glm::glm
        ${OpenCV_LIBS})
    add_dependencies(realsense_rgbd gaussian_mapper gaussian_viewer)
# endif()

if(MSVC)
    set_target_properties(simple_knn PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    set_target_properties(cuda_rasterizer PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    set_target_properties(imgui PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    set_target_properties(gaussian_mapper PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    set_target_properties(gaussian_viewer PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    set_target_properties(gaussian_viewer PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
endif()

# message(STATUS "TORCH_INCLUDE_DIRS: ${TORCH_INCLUDE_DIRS}")
message(STATUS "Boost_LIBRARIES: ${Boost_LIBRARIES}")
message(STATUS "CMAKE_LIBRARY_OUTPUT_DIRECTORY: ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")