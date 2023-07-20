cmake_minimum_required(VERSION 3.18)
cmake_policy(SET CMP0063 NEW)

project(flecs LANGUAGES C)

option(FLECS_PIC "Compile static flecs lib with position independent code (PIC)" ON)

include(cmake/target_default_compile_warnings.cmake)
include(cmake/target_default_compile_options.cmake)

# Automatically generate the same folder structure in Visual Studio as we have on disk
macro(GroupSources curdir)
	file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${curdir} ${PROJECT_SOURCE_DIR}/${curdir}/*)
	foreach(child ${children})
		if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${curdir}/${child})
			GroupSources(${curdir}/${child})
		else()
			string(REPLACE "/" "\\" groupname ${curdir})
            source_group(${groupname} FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
		endif()
	endforeach()
endmacro()

file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/ ${PROJECT_SOURCE_DIR}/*)
foreach(child ${children})
	if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${child})
		GroupSources(${child})
	endif()
endforeach()

file(GLOB_RECURSE INC include/*.h include/*.hpp)
file(GLOB_RECURSE SRC src/*.c)

set(FLECS_TARGETS "")

macro(add_flecs_target TARGET)
    add_library(${TARGET} ${INC} ${SRC})
    add_library(flecs::${TARGET} ALIAS ${TARGET})

    target_default_compile_options_c(${TARGET})
    target_default_compile_warnings_c(${TARGET})

    find_package(Thread REQUIRED)
    find_package(Socket REQUIRED)
    target_link_libraries(${TARGET} Thread::Thread Socket::Socket)

    if(WIN32)
        target_compile_definitions(${TARGET} PRIVATE _CRT_SECURE_NO_WARNINGS)
    endif()

    if(FLECS_PIC)
        set_property(TARGET ${TARGET} PROPERTY POSITION_INDEPENDENT_CODE ON)
    endif()

    target_include_directories(${TARGET} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>)

    list(APPEND FLECS_TARGETS ${TARGET})
endmacro()

add_flecs_target(flecs)

# define the install steps
include(GNUInstallDirs)
install(DIRECTORY "${PROJECT_SOURCE_DIR}/include/"
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.hpp"
        PATTERN "*.inl")

install(TARGETS ${FLECS_TARGETS}
        EXPORT flecs-export
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(EXPORT flecs-export
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/flecs
        NAMESPACE flecs::
        FILE flecs-config.cmake)
