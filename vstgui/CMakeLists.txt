##########################################################################################
cmake_minimum_required(VERSION 3.5)

if(NOT PROJECT_NAME)
    project(vstgui)
    set(VSTGUI_MAIN_PROJECT_BUILD 1)
endif()

set(CMAKE_OSX_DEPLOYMENT_TARGET 10.9)

if (UNIX AND NOT CMAKE_HOST_APPLE)
    set(LINUX TRUE)
endif()

##########################################################################################
function(vstgui_set_cxx_version target version)
    set_property(TARGET ${target} PROPERTY CXX_STANDARD ${version})
    set_property(TARGET ${target} PROPERTY CXX_STANDARD_REQUIRED ON)
    set_property(TARGET ${target} PROPERTY CMAKE_CXX_STANDARD_REQUIRED ON)
    if(APPLE)
        target_compile_options(${target} PUBLIC "-stdlib=libc++")
    endif()
endfunction(vstgui_set_cxx_version target version)

##########################################################################################
function(vstgui_source_group_by_folder target)
  if(CMAKE_CONFIGURATION_TYPES)
    set(SOURCE_GROUP_DELIMITER "/")
    set(last_dir "")
    set(files "")
    foreach(file ${${target}_sources})
      get_filename_component(dir "${file}" DIRECTORY)
      string(FIND "${dir}" ${target} isTargetFolder)
      if (${isTargetFolder} EQUAL 0)
        string(LENGTH ${target} offset)
        string(SUBSTRING "${dir}" ${offset} -1 dir)
      endif(${isTargetFolder} EQUAL 0)
      if (NOT "${dir}" STREQUAL "${last_dir}")
        if (files)
          source_group("${last_dir}" FILES ${files})
        endif (files)
        set(files "")
      endif (NOT "${dir}" STREQUAL "${last_dir}")
      set(files ${files} ${file})
      set(last_dir "${dir}")
    endforeach(file)
    if (files)
      source_group("${last_dir}" FILES ${files})
    endif (files)
  endif(CMAKE_CONFIGURATION_TYPES)
endfunction(vstgui_source_group_by_folder)

##########################################################################################
if(CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES Debug Release ReleaseLTO)
    set_property(GLOBAL PROPERTY USE_FOLDERS ON)
endif()

##########################################################################################
set(VSTGUI_COMPILE_DEFINITIONS_DEBUG "VSTGUI_LIVE_EDITING;DEBUG")
set(VSTGUI_COMPILE_DEFINITIONS_RELEASE "NDEBUG;RELEASE")
if(CMAKE_HOST_APPLE)
    if(CMAKE_C_COMPILER_VERSION VERSION_GREATER 7)
        set(VSTGUI_LTO_COMPILER_FLAGS "-O3 -flto=thin")
    else()
        set(VSTGUI_LTO_COMPILER_FLAGS "-O3 -flto")
    endif()
  set(VSTGUI_LTO_LINKER_FLAGS "")
endif()
if(LINUX)
    set(VSTGUI_LTO_COMPILER_FLAGS "-O3 -flto")
    set(VSTGUI_LTO_LINKER_FLAGS "")
    find_package(X11 REQUIRED)
    find_package(Freetype REQUIRED)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBXCB REQUIRED xcb)
    pkg_check_modules(LIBXCB_UTIL REQUIRED xcb-util)
    pkg_check_modules(LIBXCB_CURSOR REQUIRED xcb-cursor)
    pkg_check_modules(LIBXCB_KEYSYMS REQUIRED xcb-keysyms)
    pkg_check_modules(LIBXCB_XKB REQUIRED xcb-xkb)
    pkg_check_modules(LIBXKB_COMMON REQUIRED xkbcommon)
    pkg_check_modules(LIBXKB_COMMON_X11 REQUIRED xkbcommon-x11)
    set(LINUX_LIBRARIES
        ${X11_LIBRARIES}
        ${FREETYPE_LIBRARIES}
        ${LIBXCB_LIBRARIES}
        ${LIBXCB_UTIL_LIBRARIES}
        ${LIBXCB_CURSOR_LIBRARIES}
        ${LIBXCB_KEYSYMS_LIBRARIES}
        ${LIBXCB_XKB_LIBRARIES}
        ${LIBXKB_COMMON_LIBRARIES}
        ${LIBXKB_COMMON_X11_LIBRARIES}
        cairo
        fontconfig
        dl
    )
    if(VSTGUI_WARN_EVERYTHING)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-multichar")
endif()
if(MSVC)
    set(VSTGUI_COMPILE_DEFINITIONS_DEBUG "${VSTGUI_COMPILE_DEFINITIONS_DEBUG};_CRT_SECURE_NO_WARNINGS")
    set(VSTGUI_COMPILE_DEFINITIONS_RELEASE "${VSTGUI_COMPILE_DEFINITIONS_RELEASE};_CRT_SECURE_NO_WARNINGS")
    set(VSTGUI_LTO_COMPILER_FLAGS "/GL /MP")
    set(VSTGUI_LTO_LINKER_FLAGS "/LTCG")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
    set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /IGNORE:4221")
endif()

set(VSTGUI_COMPILE_DEFINITIONS PRIVATE
    $<$<CONFIG:Debug>:${VSTGUI_COMPILE_DEFINITIONS_DEBUG}>
    $<$<CONFIG:Release>:${VSTGUI_COMPILE_DEFINITIONS_RELEASE}>
    $<$<CONFIG:ReleaseLTO>:${VSTGUI_COMPILE_DEFINITIONS_RELEASE}>
)

set(CMAKE_CXX_FLAGS_RELEASELTO
    "${CMAKE_CXX_FLAGS_RELEASE} ${VSTGUI_LTO_COMPILER_FLAGS}"
)
set(CMAKE_EXE_LINKER_FLAGS_RELEASELTO
    "${CMAKE_EXE_LINKER_FLAGS_RELEASE} ${VSTGUI_LTO_LINKER_FLAGS}"
)
set(CMAKE_STATIC_LINKER_FLAGS_RELEASELTO
    "${CMAKE_STATIC_LINKER_FLAGS_RELEASE} ${VSTGUI_LTO_LINKER_FLAGS}"
)

##########################################################################################
if(NOT CMAKE_CONFIGURATION_TYPES)
    if(NOT CMAKE_BUILD_TYPE)
        set(CMAKE_BUILD_TYPE Debug)
    endif()

    message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
endif()

if(VSTGUI_MAIN_PROJECT_BUILD)
    message("Building only vstgui")
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
        $<$<CONFIG:Debug>:${CMAKE_BINARY_DIR}/Debug/>$<$<CONFIG:Release>:${CMAKE_BINARY_DIR}/Release/>$<$<CONFIG:ReleaseLTO>:${CMAKE_BINARY_DIR}/ReleaseLTO/>
    )
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY
        $<$<CONFIG:Debug>:${CMAKE_BINARY_DIR}/Debug/libs/>$<$<CONFIG:Release>:${CMAKE_BINARY_DIR}/Release/libs/>$<$<CONFIG:ReleaseLTO>:${CMAKE_BINARY_DIR}/ReleaseLTO/libs/>
    )
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
        $<$<CONFIG:Debug>:${CMAKE_BINARY_DIR}/Debug/libs>$<$<CONFIG:Release>:${CMAKE_BINARY_DIR}/Release/libs>$<$<CONFIG:ReleaseLTO>:${CMAKE_BINARY_DIR}/ReleaseLTO/libs>
    )
endif()

##########################################################################################
add_subdirectory(lib)
add_subdirectory(uidescription)

if(LINUX)
    set(VSTGUI_DISABLE_UNITTESTS  1)
endif()

if(NOT DEFINED VSTGUI_STANDALONE)
    option(VSTGUI_STANDALONE "VSTGUI Standalone library" ON)
    if(NOT DEFINED VSTGUI_STANDALONE_EXAMPLES)
      option(VSTGUI_STANDALONE_EXAMPLES "VSTGUI Standalone examples" ON)
    endif()
endif()

if(NOT VSTGUI_STANDALONE AND VSTGUI_STANDALONE_EXAMPLES)
  set(VSTGUI_STANDALONE_EXAMPLES OFF)
endif()

option(SMTG_VSTGUI_TOOLS "Build VSTGUI Tools" ON)

if(VSTGUI_STANDALONE)
    add_subdirectory(standalone)
    if(NOT VSTGUI_DISABLE_UNITTESTS)
        add_subdirectory(tests/gfxtest)
        add_subdirectory(tests/base64codecspeed)
    endif()
endif()
if(NOT VSTGUI_DISABLE_UNITTESTS)
    add_subdirectory(tests)
endif()
if(SMTG_VSTGUI_TOOLS)
    add_subdirectory(tools)
endif()

get_directory_property(hasParent PARENT_DIRECTORY)
if(hasParent)
    set(VSTGUI_COMPILE_DEFINITIONS ${VSTGUI_COMPILE_DEFINITIONS} PARENT_SCOPE)
    set(VSTGUI_LTO_COMPILER_FLAGS ${VSTGUI_LTO_COMPILER_FLAGS} PARENT_SCOPE)
endif()
