cmake_minimum_required(VERSION 3.10)

project(libsgm_ocl)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

option(BUILD_EXAMPLES "Build sgm ocl example" ON)
set(CL_TARGET_OPENCL_VERSION 120 CACHE STRING "OpenCL target version")
message(STATUS "OpenCL target verison: ${CL_TARGET_OPENCL_VERSION}")

find_package(OpenCL REQUIRED)

include(CMakeRC.cmake)

set(OCL_FILES
    src/ocl/inttypes.cl
    src/ocl/utility.cl
    src/ocl/census.cl
    src/ocl/path_aggregation_common.cl
    src/ocl/path_aggregation_vertical.cl
    src/ocl/path_aggregation_horizontal.cl
    src/ocl/path_aggregation_oblique.cl
    src/ocl/winner_takes_all.cl
    src/ocl/check_consistency.cl
    src/ocl/median_filter.cl
    src/ocl/correct_disparity_range.cl
)

cmrc_add_resource_library(rc_ocl_sgm NAMESPACE "ocl_sgm" ${OCL_FILES})

set(LIBSGM_SRCS
    src/census_transform.hpp
    src/path_aggregation.h
    src/census.cpp
    src/libsgm_ocl.cpp
    src/path_aggregation.cpp
    src/path_aggregation_vertical.cpp
    src/sgm.hpp
    src/sgm.cpp
    src/device_buffer.hpp
    src/device_kernel.h
    src/winner_takes_all.hpp
    src/winner_takes_all.cpp
    src/sgm_details.h
    src/sgm_details.cpp
)

set(LIBSGM_PUBLIC_HDRS
    libsgm_ocl/libsgm_ocl.h
    libsgm_ocl/types.h
)

set(TARGET_NAME sgm_ocl)

add_library(${TARGET_NAME} SHARED ${LIBSGM_SRCS} ${LIBSGM_PUBLIC_HDRS} ${OCL_FILES})
target_compile_definitions(${TARGET_NAME} PUBLIC -DCL_TARGET_OPENCL_VERSION=${CL_TARGET_OPENCL_VERSION})
add_library(libsgm_ocl::${TARGET_NAME} ALIAS ${TARGET_NAME})
source_group(ocl FILES ${OCL_FILES})
source_group(${TARGET_NAME} FILES ${LIBSGM_PUBLIC_HDRS})

# hack cmake bug, workaround found at https://gitlab.kitware.com/cmake/cmake/-/issues/15415#note_333595
# cannot link object library, because it requires that library to be exported
#ugly workaround
#   get_target_property(TMP cmrc-base INTERFACE_INCLUDE_DIRECTORIES)
#   target_include_directories(${TARGET_NAME} PRIVATE "${TMP}")
#little bit nicer
target_link_libraries(${TARGET_NAME} PRIVATE $<BUILD_INTERFACE:rc_ocl_sgm>)
#for debugging
find_package(OpenCV REQUIRED)
target_link_libraries(${TARGET_NAME} PRIVATE ${OpenCV_LIBRARIES})

set_target_properties(${TARGET_NAME} PROPERTIES DEBUG_POSTFIX _d RELWITHDEBINFO_POSTFIX _rd)
target_include_directories(${TARGET_NAME}
    PUBLIC 
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include/libsgm_ocl>
)
target_link_libraries(${TARGET_NAME} PUBLIC OpenCL::OpenCL)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/gen/${PROJECT_NAME}-config-version.cmake"
  VERSION 0.0.2
  COMPATIBILITY AnyNewerVersion
)

configure_file(cmake/${PROJECT_NAME}-config.cmake.in ${PROJECT_NAME}-config.cmake @ONLY)
install(TARGETS ${TARGET_NAME} EXPORT ${PROJECT_NAME}-targets DESTINATION
    ARCHIVE DESTINATION lib LIBRARY DESTINATION lib RUNTIME DESTINATION bin)
install(FILES 
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake 
        ${CMAKE_CURRENT_BINARY_DIR}/gen/${PROJECT_NAME}-config-version.cmake
    DESTINATION 
        lib/cmake/${PROJECT_NAME})
    
install(EXPORT ${PROJECT_NAME}-targets NAMESPACE libsgm_ocl:: DESTINATION lib/cmake/${PROJECT_NAME})

install(FILES ${LIBSGM_PUBLIC_HDRS} DESTINATION include/libsgm_ocl/libsgm_ocl)


if (BUILD_EXAMPLES)
    find_package(OpenCV REQUIRED)
    add_executable(stereo_movie examples/stereo_movie.cpp)
    target_link_libraries(stereo_movie PRIVATE libsgm_ocl::${TARGET_NAME} ${OpenCV_LIBRARIES})
endif()
