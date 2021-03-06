if (APPLE)
  set(OPENCL_MAIN_HEADER ${OPENCL_INCLUDE_DIR}/cl.h )
  set(CMOCK_CONFIG_FILE cmock_apple.yml )
else()
  set(OPENCL_MAIN_HEADER ${OPENCL_INCLUDE_DIR}/CL/cl.h )
  set(CMOCK_CONFIG_FILE cmock.yml )
endif()

add_custom_command(
    OUTPUT stripped/cl.h
    COMMAND ${CMAKE_COMMAND} -E make_directory stripped
    COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/strip_defines.py ${OPENCL_MAIN_HEADER} stripped/cl.h
    COMMENT "Stripping defines from cl.h"
    DEPENDS ${OPENCL_MAIN_HEADER} strip_defines.py)
add_library(stripped_cl_h OBJECT stripped/cl.h)

add_custom_command(
    OUTPUT mocks/Mockcl.c mocks/Mockcl.h
	COMMAND ${CMAKE_COMMAND} -E make_directory mocks
    COMMAND ruby ${CMOCK_DIR}/lib/cmock.rb -o${CMAKE_CURRENT_SOURCE_DIR}/${CMOCK_CONFIG_FILE} stripped/cl.h
    COMMENT "Generating mocks"
    DEPENDS stripped/cl.h ${CMOCK_CONFIG_FILE})
add_library(mock_cl STATIC mocks/Mockcl.c mocks/Mockcl.h)
add_dependencies(mock_cl stripped_cl_h)
	
add_custom_command(
    OUTPUT test_clhpp_Runner.c
    COMMAND ruby ${UNITY_DIR}/auto/generate_test_runner.rb test_clhpp.cpp ${CMOCK_CONFIG_FILE} ${CMAKE_CURRENT_BINARY_DIR}/test_clhpp_Runner.c
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating test runner"
    DEPENDS test_clhpp.cpp ${CMOCK_CONFIG_FILE})
add_library(test_clhpp_runner STATIC test_clhpp_Runner.c test_clhpp.cpp)
target_link_libraries(test_clhpp_runner mock_cl)
add_dependencies(test_clhpp_runner mock_cl)

if (SUPPORTS_OPEN_CL_2_0_APIS OR SUPPORTS_OPEN_CL_2_1_APIS OR SUPPORTS_OPEN_CL_2_2_APIS)
  add_custom_command(
      OUTPUT test_cl2hpp_Runner.c
      COMMAND ruby ${UNITY_DIR}/auto/generate_test_runner.rb test_cl2hpp.cpp ${CMOCK_CONFIG_FILE} ${CMAKE_CURRENT_BINARY_DIR}/test_cl2hpp_Runner.c
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
      COMMENT "Generating test runner for 2.0"
      DEPENDS test_cl2hpp.cpp ${CMOCK_CONFIG_FILE})
  add_library(test_cl2hpp_runner STATIC test_cl2hpp_Runner.c test_cl2hpp.cpp test_clhpp.cpp)
  target_link_libraries(test_cl2hpp_runner mock_cl)
  add_dependencies(test_cl2hpp_runner mock_cl)
endif()

include_directories(
    ${CLCPP_BINARY_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}/mocks
    ${OPENCL_DIST_DIR}/include
    ${OPENCL_INCLUDE_DIR}
    ${OPENCL_INCLUDE_DIR}/CL
    ${UNITY_DIR}/src
    ${CMOCK_DIR}/src)

add_definitions(-DUNITY_SUPPORT_64)
if( CMAKE_SIZEOF_VOID_P EQUAL 8 )
    add_definitions(-DUNITY_POINTER_WIDTH=64)
    add_definitions("-DCMOCK_MEM_PTR_AS_INT=unsigned long long")
    add_definitions(-DCMOCK_MEM_ALIGN=3)
endif()
if( CMAKE_SIZEOF_LONG EQUAL 8 )
    add_definitions(-DUNITY_LONG_WIDTH=64)
endif()

set(TEST_CLHPP_HEADERS
    ${CLCPP_BINARY_DIR}/include/CL/cl.hpp
    mocks/Mockcl.h)
set_source_files_properties(${CLCPP_BINARY_DIR}/include/CL/cl.hpp PROPERTIES GENERATED 1)

if (SUPPORTS_OPEN_CL_2_0_APIS OR SUPPORTS_OPEN_CL_2_1_APIS OR SUPPORTS_OPEN_CL_2_2_APIS)
  set(TEST_CL2HPP_HEADERS
      ${CLCPP_BINARY_DIR}/include/CL/cl2.hpp
      mocks/Mockcl.h)
  set_source_files_properties(${CLCPP_BINARY_DIR}/include/CL/cl2.hpp PROPERTIES GENERATED 1)
endif()

set(TEST_CLHPP_SOURCES_COMMON
    ${CMAKE_CURRENT_BINARY_DIR}/mocks/Mockcl.c
    ${CMOCK_DIR}/src/cmock.c
    ${UNITY_DIR}/src/unity.c)

set(TEST_CLHPP_SOURCES
    ${TEST_CLHPP_SOURCES_COMMON}
)

if (SUPPORTS_OPEN_CL_2_0_APIS OR SUPPORTS_OPEN_CL_2_1_APIS OR SUPPORTS_OPEN_CL_2_2_APIS)
  set(TEST_CL2HPP_SOURCES
      ${TEST_CLHPP_SOURCES_COMMON}
      ${CMAKE_CURRENT_BINARY_DIR}/test_cl2hpp_Runner.c
      test_cl2hpp.cpp)
endif()
    
set_source_files_properties(test_clhpp_Runner.c PROPERTIES LANGUAGE "C")

set(TEST_LIBS "")
if (WIN32)
  list(APPEND TEST_LIBS opengl32)
elseif (APPLE)
  find_library(cocoa_library Cocoa)
  find_library(opengl_library OpenGL)
  find_library(corevideo_library CoreVideo)
  find_library(iokit_library IOKit)
  find_library(opencl_library OpenCL)
  find_library(system_library System)
  list(APPEND TEST_LIBS c++ ${system_library} ${cocoa_library} ${opengl_library} ${corevideo_library} ${iokit_library} ${opencl_library})
elseif(CMAKE_SYSTEM MATCHES "Linux")
  list(APPEND TEST_LIBS GL Xxf86vm Xrandr Xinerama Xcursor Xi X11 pthread dl rt)
endif()

#  message(FATAL_ERROR "TEST_CLHPP_HEADERS = \n${TEST_CLHPP_HEADERS}\n\nTEST_CLHPP_SOURCES = \n${TEST_CLHPP_SOURCES}")

add_executable(test_clhpp test_clhpp.cpp ${TEST_CLHPP_SOURCES} ${TEST_CLHPP_HEADERS})
target_link_libraries(test_clhpp test_clhpp_runner)
add_dependencies(test_clhpp generate_clhpp test_clhpp_runner)
add_test(run_test_clhpp test_clhpp)

add_executable(test_clhpp_deprecated_1_1 ${TEST_CLHPP_SOURCES} ${TEST_CLHPP_HEADERS})
target_link_libraries(test_clhpp_deprecated_1_1 ${TEST_LIBS} test_clhpp_runner)
set_target_properties(test_clhpp_deprecated_1_1 PROPERTIES COMPILE_DEFINITIONS "CL_USE_DEPRECATED_OPENCL_1_1_APIS")
add_dependencies(test_clhpp_deprecated_1_1 generate_clhpp test_clhpp_runner)
add_test(run_test_clhpp_deprecated_1_1 test_clhpp_deprecated_1_1)

add_executable(test_clhpp_deprecated_1_2 ${TEST_CLHPP_SOURCES} ${TEST_CLHPP_HEADERS})
target_link_libraries(test_clhpp_deprecated_1_2 ${TEST_LIBS} test_clhpp_runner)
set_target_properties(test_clhpp_deprecated_1_2 PROPERTIES COMPILE_DEFINITIONS "CL_USE_DEPRECATED_OPENCL_1_1_APIS")
set_target_properties(test_clhpp_deprecated_1_2 PROPERTIES COMPILE_DEFINITIONS "CL_USE_DEPRECATED_OPENCL_1_2_APIS")
add_dependencies(test_clhpp_deprecated_1_2 generate_clhpp test_clhpp_runner)
add_test(run_test_clhpp_deprecated_1_2 test_clhpp_deprecated_1_2)

if (SUPPORTS_OPEN_CL_2_0_APIS OR SUPPORTS_OPEN_CL_2_1_APIS OR SUPPORTS_OPEN_CL_2_2_APIS)
  set_source_files_properties(test_cl2hpp_Runner.c PROPERTIES LANGUAGE "C")
  add_executable(test_cl2hpp ${TEST_CL2HPP_SOURCES} ${TEST_CL2HPP_HEADERS})
  target_link_libraries(test_cl2hpp test_cl2hpp_runner )
  add_dependencies(test_cl2hpp generate_cl2hpp test_cl2hpp_runner)
  add_test(run_test_cl2hpp test_cl2hpp)
endif()

if( CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" )
    set(TEST_CLHPP_CXX11_SOURCES
        ${TEST_CLHPP_SOURCES_COMMON}
        ${CMAKE_CURRENT_BINARY_DIR}/test_clhpp_Runner.c
        test_clhpp_cxx11.cpp)
    set_source_files_properties(test_clhpp_cxx11.cpp PROPERTIES COMPILE_FLAGS "-std=c++11")
    if (SUPPORTS_OPEN_CL_2_0_APIS OR SUPPORTS_OPEN_CL_2_1_APIS OR SUPPORTS_OPEN_CL_2_2_APIS)
      set_source_files_properties(test_cl2hpp.cpp PROPERTIES COMPILE_FLAGS "-std=c++11")
    endif()
    add_executable(test_clhpp_cxx11 ${TEST_CLHPP_CXX11_SOURCES} ${TEST_CLHPP_HEADERS})
    add_dependencies(test_clhpp_cxx11 generate_clhpp test_clhpp_runner)
    add_test(run_test_clhpp_cxx11 test_clhpp_cxx11)
endif()
