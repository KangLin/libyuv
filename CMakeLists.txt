# CMakeLists for libyuv
# Originally created for "roxlu build system" to compile libyuv on windows
# Run with -DTEST=ON to build unit tests

PROJECT ( YUV C CXX )	# "C" is required even for C++ projects
CMAKE_MINIMUM_REQUIRED( VERSION 2.8 )
OPTION( TEST "Built unit tests" OFF )

if (BUILD_SHARED_LIBS)
    if (CMAKE_COMPILER_IS_GNUCXX AND NOT MINGW)
       # Just setting CMAKE_POSITION_INDEPENDENT_CODE should be enough to set
       # -fPIC for GCC but sometimes it still doesn't get set, so make sure it
       # does.
       add_definitions("-fPIC")
    endif()
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

SET ( ly_base_dir	${PROJECT_SOURCE_DIR} )
SET ( ly_src_dir	${ly_base_dir}/source )
SET ( ly_inc_dir	${ly_base_dir}/include )
SET ( ly_tst_dir	${ly_base_dir}/unit_test )

FILE ( GLOB	ly_header_files
    ${ly_inc_dir}/libyuv/*.h )

FILE ( GLOB_RECURSE	ly_source_files ${ly_src_dir}/*.cc )
LIST ( SORT			ly_source_files )

FILE ( GLOB_RECURSE	ly_unittest_sources ${ly_tst_dir}/*.cc )
LIST ( SORT			ly_unittest_sources )

INCLUDE_DIRECTORIES( BEFORE ${ly_inc_dir} )

ADD_LIBRARY( ${PROJECT_NAME} ${ly_source_files} )

INCLUDE ( FindJPEG )
if (JPEG_FOUND)
  target_link_libraries( ${PROJECT_NAME} PUBLIC ${JPEG_LIBRARY} )
  target_compile_definitions(${PROJECT_NAME} PUBLIC HAVE_JPEG)
  target_include_directories(${PROJECT_NAME} PUBLIC ${JPEG_INCLUDE_DIR})
endif()

# this creates the conversion tool
ADD_EXECUTABLE			( yuvconvert ${ly_base_dir}/util/yuvconvert.cc )
TARGET_LINK_LIBRARIES	( yuvconvert ${PROJECT_NAME} )

if(TEST)
  find_library(GTEST_LIBRARY gtest)
  if(GTEST_LIBRARY STREQUAL "GTEST_LIBRARY-NOTFOUND")
    set(GTEST_SRC_DIR /usr/src/gtest CACHE STRING "Location of gtest sources")
    if(EXISTS ${GTEST_SRC_DIR}/src/gtest-all.cc)
      message(STATUS "building gtest from sources in ${GTEST_SRC_DIR}")
      set(gtest_sources ${GTEST_SRC_DIR}/src/gtest-all.cc)
      add_library(gtest STATIC ${gtest_sources})
      include_directories(${GTEST_SRC_DIR})
      include_directories(${GTEST_SRC_DIR}/include)
      set(GTEST_LIBRARY gtest)
    else()
      message(FATAL_ERROR "TEST is set but unable to find gtest library")
    endif()
  endif()

  add_executable(libyuv_unittest ${ly_unittest_sources})
  target_link_libraries(libyuv_unittest ${PROJECT_NAME} ${GTEST_LIBRARY})
  find_library(PTHREAD_LIBRARY pthread)
  if(NOT PTHREAD_LIBRARY STREQUAL "PTHREAD_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest pthread)
  endif()
  if (JPEG_FOUND)
    target_link_libraries(libyuv_unittest ${JPEG_LIBRARY})
  endif()

  if(NACL AND NACL_LIBC STREQUAL "newlib")
    target_link_libraries(libyuv_unittest glibc-compat)
  endif()

  find_library(GFLAGS_LIBRARY gflags)
  if(NOT GFLAGS_LIBRARY STREQUAL "GFLAGS_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest gflags)
    add_definitions(-DLIBYUV_USE_GFLAGS)
  endif()
endif()

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

# Install head files
set_property(
    TARGET
        ${PROJECT_NAME}
    PROPERTY
        PUBLIC_HEADER ${ly_header_files}
)

# install the conversion tool, libyuv library and all the header files
INSTALL ( TARGETS yuvconvert DESTINATION ${CMAKE_INSTALL_BINDIR} )
INSTALL ( TARGETS ${PROJECT_NAME} 
    EXPORT ${PROJECT_NAME}Config
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_BINDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libyuv
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
# Install header file
INSTALL(FILES ${ly_inc_dir}/libyuv.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
# Install cmake configure files
INSTALL(
        EXPORT ${PROJECT_NAME}Config
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake"
)

# create the .deb and .rpm packages using cpack
INCLUDE ( CM_linux_packages.cmake )

WRITE_BASIC_PACKAGE_VERSION_FILE(
    "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${YUV_VERSION}
    COMPATIBILITY AnyNewerVersion)
INSTALL(FILES "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake")
