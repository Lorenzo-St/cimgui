cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0091 NEW)
project(cimgui)

set (CMAKE_CXX_STANDARD 11)

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends)
	set(BAKENDS_FOLDER "imgui/backends/")
else()
  message(FATAL_ERROR "Cannot find imgui backends")
endif()

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/imgui/imgui_tables.cpp)
	set(TABLES_SOURCE "imgui/imgui_tables.cpp")
else()
	set(TABLES_SOURCE "")
endif()


#general settings
file(GLOB IMGUI_SOURCES
    cimgui.cpp
    imgui/imgui.cpp
    imgui/imgui_draw.cpp
    imgui/imgui_demo.cpp
    imgui/imgui_widgets.cpp
	${TABLES_SOURCE}
)

set(IMGUI_STATIC "no" CACHE STRING "Build as a static library")
set(IMGUI_FREETYPE "no" CACHE STRING "Build with freetype library")
set(IMGUI_LIBRARIES )
set(IMGUI_SOURCES_sdl)
set(IMGUI_LIBRARIES )

if(IMGUI_FREETYPE)
	message(STATUS "Building with freetype")
	FIND_PACKAGE(freetype REQUIRED PATHS ${FREETYPE_PATH})
	list(APPEND IMGUI_LIBRARIES freetype)
	list(APPEND IMGUI_SOURCES imgui/misc/freetype/imgui_freetype.cpp)
endif(IMGUI_FREETYPE)


add_compile_definitions("IMGUI_IMPL_OPENGL_LOADER_GL3W")

#opengl3
list(APPEND IMGUI_SOURCES ${BAKENDS_FOLDER}imgui_impl_opengl3.cpp)
if(WIN32)
       list(APPEND IMGUI_LIBRARIES opengl32)
else(WIN32)#Unix
       list(APPEND IMGUI_LIBRARIES GL)
endif(WIN32)


#sdl2
list(APPEND IMGUI_SOURCES ${BAKENDS_FOLDER}imgui_impl_sdl2.cpp)
if(DEFINED SDL_PATH)
	message(STATUS "SDL_PATH defined as " ${SDL_PATH})
	FIND_PACKAGE(SDL2 PATHS ${SDL_PATH})
else(DEFINED SDL_PATH)
	# If SDL_PATH is not set, fallback and attempt to find SDL cmake script at a default location
	find_package(SDL2)
endif(DEFINED SDL_PATH)

if(SDL2_FOUND)
	get_target_property(SDL_INCLUDE SDL2::SDL2 INTERFACE_INCLUDE_DIRECTORIES)
	message(STATUS "sdlinclude is " ${SDL_INCLUDE})
	if ("${SDL_INCLUDE}" STREQUAL "" OR "${SDL_INCLUDE}" STREQUAL "SDL_INCLUDE-NOTFOUND") #if not found latest SDL2 cmake config use older
		message(STATUS "sdlinclude2 is " ${SDL2_INCLUDE_DIRS})
		include_directories(${SDL2_INCLUDE_DIRS})
		set(IMGUI_SDL_LIBRARY ${SDL2_LIBRARIES})
		message(STATUS IMGUI_SDL_LIBRARY ${SDL2_LIBRARIES})
	else()#use new one SDL2 config
		include_directories(${SDL_INCLUDE})
		set(IMGUI_SDL_LIBRARY SDL2::SDL2)
		set(SDL_MAIN SDL2::SDL2main)
		message(STATUS ${SDL_MAIN} ${IMGUI_SDL_LIBRARY})
	endif()
else(SDL2_FOUND)
	if(DEFINED SDL_PATH)
		message(FATAL_ERROR "Cannot find SDL at SDL_PATH")
	else(DEFINED SDL_PATH)
		message(FATAL_ERROR "Cannot find SDL. Maybe try specifying SDL_PATH?")
	endif(DEFINED SDL_PATH)
endif(SDL2_FOUND)

add_library(cimgui_sdl SHARED ${IMGUI_SOURCES})
target_link_libraries(cimgui_sdl ${IMGUI_LIBRARIES} ${IMGUI_SDL_LIBRARY})

#using library
include_directories(generator/output/)
include_directories(imgui)


#add library and link
if (IMGUI_STATIC)
    add_library(cimgui STATIC ${IMGUI_SOURCES})
else (IMGUI_STATIC)
    add_library(cimgui SHARED ${IMGUI_SOURCES})
endif (IMGUI_STATIC)

if(IMGUI_FREETYPE)
	target_compile_definitions(cimgui PUBLIC IMGUI_ENABLE_FREETYPE)
	target_compile_definitions(cimgui PUBLIC IMGUI_ENABLE_STB_TRUETYPE)
endif(IMGUI_FREETYPE)

if(IMGUI_WCHAR32)
	target_compile_definitions(cimgui PUBLIC IMGUI_USE_WCHAR32)
endif(IMGUI_WCHAR32)

add_definitions("-DIMGUI_USER_CONFIG=\"../cimconfig.h\"")
target_compile_definitions(cimgui PUBLIC IMGUI_DISABLE_OBSOLETE_FUNCTIONS=1)
if (WIN32)
    target_compile_definitions(cimgui PUBLIC IMGUI_IMPL_API=extern\t\"C\"\t__declspec\(dllexport\))
    list(APPEND IMGUI_LIBRARIES imm32)
else (WIN32)
    target_compile_definitions(cimgui PUBLIC IMGUI_IMPL_API=extern\t\"C\"\t)
endif (WIN32)


target_include_directories(cimgui PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(cimgui PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/imgui)
set_target_properties(cimgui PROPERTIES PREFIX "")
target_link_libraries(cimgui ${IMGUI_LIBRARIES})
target_link_libraries(cimgui ${IMGUI_SDL_LIBRARY} cimgui_sdl)

#install
install(TARGETS cimgui
    RUNTIME DESTINATION  .
    LIBRARY DESTINATION  .
    ARCHIVE DESTINATION  .
)

#test
set(CIMGUI_TEST "no" CACHE STRING "Enable compilation of a test unit based on imgui null")

if (CIMGUI_TEST)
  add_subdirectory(test)
endif ()
