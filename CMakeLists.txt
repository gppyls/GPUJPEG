cmake_minimum_required(VERSION 3.8.0 FATAL_ERROR)
project(gpujpeg LANGUAGES C CXX CUDA)

set(CPACK_PACKAGE_INSTALL_DIRECTORY "gpujpeg") # may be removed when version will be explicitly defined
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
include(CPack)

execute_process(COMMAND git rev-parse --short HEAD
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        OUTPUT_VARIABLE GIT_REV
        OUTPUT_STRIP_TRAILING_WHITESPACE)
add_definitions(-DGIT_REV=\"${GIT_REV}\")

# Include cmake modules in "cmake" directory (FindGLEW.cmake and FindGLUT.cmake)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

# Common settings
include_directories(${CMAKE_SOURCE_DIR})

set(NEEDED_CXX_FEATURES cxx_delegating_constructors cxx_nullptr)

# Find OpenGL, GLEW and GLUT
find_package(OpenGL)
find_package(GLEW)
find_package(GLUT)
if(NOT OPENGL_FOUND)
    set(OPENGL_ERROR "OpenGL library not found")
elseif(NOT GLEW_FOUND)
    set(OPENGL_ERROR "GLEW library not found")
elseif(NOT GLUT_FOUND)
    set(OPENGL_ERROR "GLUT not found")
elseif(NOT GLEW_INCLUDE_DIR)
    set(OPENGL_ERROR "GLEW header files not found")
elseif(NOT GLUT_INCLUDE_DIR)
    set(OPENGL_ERROR "GLUT header files not found")
elseif(NOT OPENGL_INCLUDE_DIR)
    # On Windows the FindOpenGL.cmake doesn't work right for OPENGL_INCLUDE_DIR and thus reuse GLEW_INCLUDE_DIR
    if(MSVC)
        set(OPENGL_INCLUDE_DIR "${GLEW_INCLUDE_DIR}")
    else()
        set(OPENGL_ERROR "OpenGL header files not found")
    endif()
endif()
if(MSVC AND NOT GLEW_DLL)
    set(OPENGL_ERROR "GLEW DLL not found")
elseif(MSVC AND NOT GLUT_DLL)
    set(OPENGL_ERROR "GLUT DLL not found")
endif()
if(NOT OPENGL_ERROR)
    message(STATUS "OpenGL: ${OPENGL_gl_LIBRARY} (${OPENGL_INCLUDE_DIR})")
    message(STATUS "GLU: ${OPENGL_glu_LIBRARY}")
    message(STATUS "GLEW: ${GLEW_LIBRARIES} (${GLEW_INCLUDE_DIRS})")
    message(STATUS "GLUT: ${GLUT_glut_LIBRARY} (${GLUT_INCLUDE_DIR})")

    # OpenGL executables are enabled
    set(GPUJPEG_OPENGL_ENABLED TRUE)

    # Build GPUJPEG library with OpenGL support
    add_definitions("-DGPUJPEG_USE_OPENGL")
    include_directories("${OPENGL_INCLUDE_DIR}" "${GLEW_INCLUDE_DIRS}" "${GLUT_INCLUDE_DIR}")

    # Prepare list of OpenGL libraries for linking
    set(GPUJPEG_OPENGL_LIBRARIES)
    list(APPEND GPUJPEG_OPENGL_LIBRARIES ${GLEW_LIBRARIES})
    list(APPEND GPUJPEG_OPENGL_LIBRARIES ${GLUT_glut_LIBRARY})
    list(APPEND GPUJPEG_OPENGL_LIBRARIES ${OPENGL_LIBRARIES})
endif()

# GPUJPEG library
file(GLOB H_FILES libgpujpeg/*.h)
file(GLOB C_FILES src/*.c src/*/*.cpp src/*.cu)
add_library(gpujpeg SHARED ${H_FILES} ${C_FILES})
target_compile_features(gpujpeg PRIVATE ${NEEDED_CXX_FEATURES})
set_property(TARGET gpujpeg PROPERTY C_STANDARD 99)
set_target_properties(gpujpeg PROPERTIES SOVERSION 0)
SET_TARGET_PROPERTIES(
  gpujpeg
PROPERTIES
  PUBLIC_HEADER "${H_FILES}"
)
if(GPUJPEG_OPENGL_ENABLED)
    target_link_libraries(gpujpeg ${GPUJPEG_OPENGL_LIBRARIES})
endif()

# GPUJPEG commmand-line application
file(GLOB C_FILES src/main.c)
add_executable(gpujpeg-bin ${C_FILES})
target_compile_features(gpujpeg-bin PRIVATE ${NEEDED_CXX_FEATURES})
set_property(TARGET gpujpeg-bin PROPERTY C_STANDARD 99)
target_link_libraries(gpujpeg-bin gpujpeg)
set_target_properties(gpujpeg-bin PROPERTIES OUTPUT_NAME "gpujpeg")

install(TARGETS gpujpeg gpujpeg-bin
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        PUBLIC_HEADER DESTINATION include/libgpujpeg
        RUNTIME DESTINATION bin)

install(FILES AUTHORS CONTRIBUTING.md COPYING FAQ.md INSTALL NEWS README.md
        DESTINATION doc)

# When OpenGL was found, include OpenGL executables
if(GPUJPEG_OPENGL_ENABLED)

    # GPUJPEG decoder to OpenGL texture
    file(GLOB FILES test/decoder_gltex/*)
    add_executable(decoder_gltex ${FILES})
    target_compile_features(decoder_gltex PRIVATE ${NEEDED_CXX_FEATURES})
    set_property(TARGET decoder_gltex PROPERTY C_STANDARD 99)
    target_link_libraries(decoder_gltex gpujpeg)

    # OpenGL interoperability example (currently not working)
    #file(GLOB FILES test/opengl_interop/*.c test/opengl_interop/*.h test/opengl_interop/*.cu)
    #cuda_add_executable(opengl_interop ${FILES})
    #target_link_libraries(opengl_interop src)

endif()

add_custom_target(unittests make -C test/unit LIBDIR=${CMAKE_BINARY_DIR}
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        DEPENDS gpujpeg)

add_custom_target(tests
        DEPENDS unittests)

add_custom_target(cscope cscope -bv `find ${CMAKE_SOURCE_DIR}/libgpujpeg ${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/test -type f \( -name '*.c' -o -name '*.cpp' -o -name '*.cu' -o -name '*.h' -o -name '*.hpp' -o -name '*.m' -o -name '*.mm' \)`)
add_custom_target(universal-ctags ctags-universal `find ${CMAKE_SOURCE_DIR}/libgpujpeg ${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/test -type f \( -name '*.c' -o -name '*.cpp' -o -name '*.cu' -o -name '*.h' -o -name '*.hpp' -o -name '*.m' -o -name '*.mm' \)`)

