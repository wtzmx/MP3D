cmake_minimum_required(VERSION 3.16)
project(Matterport_Simulator LANGUAGES CXX)

# 设置CMake策略以避免警告
if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
    cmake_policy(SET CMP0135 NEW)
endif()

# 设置C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 选项
option(OSMESA_RENDERING "Offscreen CPU rendering with OSMesa" OFF)
option(EGL_RENDERING "Offscreen GPU rendering with EGL" OFF)

# 设置构建类型
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# 添加自定义模块路径
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/Modules")

# 包含目录
include_directories("${PROJECT_SOURCE_DIR}/include")

# 查找依赖包
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})

find_package(PkgConfig REQUIRED)
find_package(OpenMP)
if (OpenMP_CXX_FOUND)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
endif()

pkg_check_modules(JSONCPP REQUIRED jsoncpp)

# 处理渲染选项
if(EGL_RENDERING)
  add_definitions(-DEGL_RENDERING)
  find_package(OpenGL REQUIRED COMPONENTS OpenGL EGL)
  pkg_check_modules(EPOXY REQUIRED epoxy)
  set(GL_LIBS OpenGL::OpenGL OpenGL::EGL ${EPOXY_LIBRARIES})
elseif(OSMESA_RENDERING)
  add_definitions(-DOSMESA_RENDERING)
  pkg_check_modules(OSMESA REQUIRED osmesa)
  set(GL_LIBS ${OSMESA_LIBRARIES})
else()
  find_package(OpenGL REQUIRED)
  find_package(GLEW REQUIRED)
  set(GL_LIBS OpenGL::GL GLEW::GLEW)
endif()

# 主库
add_library(MatterSim SHARED
  src/lib/MatterSim.cpp
  src/lib/NavGraph.cpp
  src/lib/Benchmark.cpp
  src/lib/cbf.cpp
)
target_include_directories(MatterSim PRIVATE ${JSONCPP_INCLUDE_DIRS})
target_link_libraries(MatterSim PRIVATE ${JSONCPP_LIBRARIES} ${OpenCV_LIBS} ${GL_LIBS})
if(OSMESA_RENDERING)
  target_compile_definitions(MatterSim PUBLIC OSMESA_RENDERING)
endif()

# 测试可执行文件
add_executable(tests src/test/main.cpp)
target_include_directories(tests PRIVATE ${JSONCPP_INCLUDE_DIRS})
target_link_libraries(tests PRIVATE MatterSim ${JSONCPP_LIBRARIES} ${OpenCV_LIBS})

# 主程序可执行文件
add_executable(mattersim_main src/driver/mattersim_main.cpp)
target_link_libraries(mattersim_main PRIVATE MatterSim ${OpenCV_LIBS})
target_include_directories(mattersim_main PRIVATE ${OpenCV_INCLUDE_DIRS})

# Python绑定
set(PYBIND11_PYTHON_VERSION 3)
set(PYBIND11_FINDPYTHON ON)
add_subdirectory(pybind11)

find_package(Python3 COMPONENTS Interpreter Development NumPy REQUIRED)

pybind11_add_module(MatterSimPython src/lib_python/MatterSimPython.cpp)
target_include_directories(MatterSimPython PRIVATE ${Python3_NumPy_INCLUDE_DIRS})
target_link_libraries(MatterSimPython PRIVATE MatterSim)
set_target_properties(MatterSimPython PROPERTIES OUTPUT_NAME MatterSim)