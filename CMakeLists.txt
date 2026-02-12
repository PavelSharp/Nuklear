cmake_minimum_required(VERSION 3.29 FATAL_ERROR)
if (POLICY CMP0141)
  cmake_policy(SET CMP0141 NEW)
  set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<IF:$<AND:$<C_COMPILER_ID:MSVC>,$<CXX_COMPILER_ID:MSVC>>,$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>,$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>>")
endif()
if ("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "In-source builds are not allowed.")
endif()
if (NOT DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type")
endif()

set(frontend "${CMAKE_CXX_COMPILER_FRONTEND_VARIANT}")

function(nk_add_sanitizer tgt)
	if (frontend STREQUAL "GNU")
		target_compile_options(${tgt} PRIVATE
			-fsanitize=address,undefined
			-fno-omit-frame-pointer
			-g
		)
		target_link_options(${tgt} PRIVATE -fsanitize=address,undefined)
    elseif (frontend STREQUAL "MSVC")
		target_compile_options(${tgt} PRIVATE /fsanitize=address)
    else()
		message(WARNING "add_sanitizer(${tgt}): unknown frontend variant: ${frontend}")
	endif()
endfunction()

function(nk_add_warnings tgt)
	if (frontend STREQUAL "GNU")
		target_compile_options(${tgt} PRIVATE -Wall -Wextra -pedantic)
    elseif (frontend STREQUAL "MSVC")
		target_compile_options(${tgt} PRIVATE /W4)
    else()
		message(WARNING "add_warnings(${tgt}): unknown frontend variant: ${frontend}")
	endif()
endfunction()

set(INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include" "${CMAKE_SOURCE_DIR}" "${SRC_DIR}")

#find_package(GLEW REQUIRED)
#find_package(glfw3 CONFIG REQUIRED)
#find_path(DIRENT_INCLUDE_DIRS "dirent.h")

function(nk_add_executable tgt src)
    add_executable (${tgt} MACOSX_BUNDLE "${src}")
    set_target_properties(${tgt} PROPERTIES WIN32_EXECUTABLE $<CONFIG:Release>)
    if (frontend STREQUAL "MSVC")
        target_link_options(${tgt} PRIVATE /ENTRY:mainCRTStartup>)
    endif()

    if (EMSCRIPTEN)
      set_target_properties(${tgt} PROPERTIES OUTPUT_NAME "index" SUFFIX ".html")
      target_link_options(${tgt} PRIVATE "-sEXPORTED_RUNTIME_METHODS=ccall") #,FS  "--shell-file=${CMAKE_SOURCE_DIR}/shell.html"
    endif()
    if (NOT EMSCRIPTEN)
    add_custom_command(TARGET ${tgt} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy -t $<TARGET_FILE_DIR:${tgt}> $<TARGET_RUNTIME_DLLS:${tgt}>
        COMMAND_EXPAND_LISTS
    )
    endif()
endfunction()

include(FetchContent)
function(nk_require_sdl2)
    set(SDL_TEST OFF CACHE BOOL "" FORCE)
    set(SDL_TESTS OFF CACHE BOOL "" FORCE)
    FetchContent_Declare(
        sdl2
        GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
        GIT_TAG release-2.30.x
        GIT_SHALLOW     TRUE
    )
    FetchContent_MakeAvailable(sdl2)
endfunction()
function(nk_require_sdl3)
    FetchContent_Declare(
        sdl3
        GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
        GIT_TAG main
	    GIT_SHALLOW     TRUE
    )
    FetchContent_MakeAvailable(sdl3)
endfunction()
function(nk_require_glfw)
    if (EMSCRIPTEN)
        if (NOT TARGET glfw)
            add_library(glfw INTERFACE)
            target_compile_options(glfw INTERFACE "--use-port=contrib.glfw3")
            target_link_options(glfw INTERFACE "--use-port=contrib.glfw3")
        endif()
        return()
    endif()
    set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
    set(GLFW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
    set(GLFW_BUILD_DOCS OFF CACHE BOOL "" FORCE)
    FetchContent_Declare(
        glfw
        GIT_REPOSITORY https://github.com/glfw/glfw.git
        GIT_TAG 3.3.8
        GIT_SHALLOW     TRUE
    )
    FetchContent_MakeAvailable(glfw)
endfunction()
function(nk_require_glew)
    if (EMSCRIPTEN)
        if (NOT TARGET glew)
            add_library(glew INTERFACE)
        endif()
        if (NOT TARGET GLEW::glew)
            add_library(GLEW::glew ALIAS glew)
        endif()
        if (NOT TARGET GLEW::glew_s)
            add_library(GLEW::glew_s ALIAS glew)
        endif()
        if (NOT TARGET GLEW::GLEW)
            add_library(GLEW::GLEW ALIAS glew)
        endif()
        return()
    endif()

    find_package(GLEW QUIET)
    if (GLEW_FOUND)
        return()
    endif()
    FetchContent_Declare(
        glew
        URL https://github.com/nigels-com/glew/releases/download/glew-2.3.1/glew-2.3.1.tgz
        SOURCE_SUBDIR glew-2.3.1
    )
    #FetchContent_MakeAvailable(glew)
    FetchContent_GetProperties(glew)
    if (NOT glew_POPULATED)
        FetchContent_Populate(glew)
        add_subdirectory(
            ${glew_SOURCE_DIR}/build/cmake
            ${glew_BINARY_DIR}
        )        
    endif()
    if (TARGET glew AND NOT TARGET GLEW::glew)
        target_include_directories(glew INTERFACE $<BUILD_INTERFACE:${glew_SOURCE_DIR}/include>)
        add_library(GLEW::glew ALIAS glew)
    endif()
    if (TARGET glew_s AND NOT TARGET GLEW::glew_s)
        target_include_directories(glew_s INTERFACE $<BUILD_INTERFACE:${glew_SOURCE_DIR}/include>)
        add_library(GLEW::glew_s ALIAS glew_s)
    endif()
endfunction()

function(nk_add_demo_sdl2_opengl3)
    nk_require_glew()
    nk_require_sdl2()
    set(tgt demo_sdl2_opengl3)
    nk_add_executable(${tgt} "demo/sdl_opengl3/main.c")
    target_link_libraries(${tgt} PRIVATE GLEW::glew SDL2::SDL2 SDL2::SDL2main)
endfunction()

function(nk_add_demo_sdl2_renderer)
    nk_require_sdl2()
    set(tgt demo_sdl2_renderer)
    nk_add_executable(${tgt} "demo/sdl_renderer/main.c")
    target_link_libraries(${tgt} PRIVATE SDL2::SDL2 SDL2::SDL2main)
endfunction()

function(nk_add_demo_sdl3_renderer)
    nk_require_sdl3()
    set(tgt demo_sdl3_renderer)
    nk_add_executable(${tgt} "demo/sdl3_renderer/main.c")
    target_link_libraries(${tgt} PRIVATE SDL3::SDL3)
endfunction()

function(nk_add_glfw_example exname out_target)
    nk_require_glew()
    nk_require_glfw()

    set(tgt "example_glfw_${exname}")
    nk_add_executable(${tgt} "example/${exname}.c")
    target_link_libraries(${tgt} PRIVATE GLEW::glew glfw)
    set(${out_target} ${tgt} PARENT_SCOPE)
endfunction()

function(nk_add_glfw_examples)
    nk_add_glfw_example("canvas" tgt_canvas)
    #target_link_options(example_glfw_canvas PRIVATE
    #    "-sUSE_WEBGL2=1"
    #    "-sFULL_ES3=1"
    #)

    #nk_add_glfw_example("extended" tgt_extended)
    #nk_add_glfw_example("skinning" tgt_skinning)

    #add_custom_command(TARGET ${tgt_extended} POST_BUILD
    #    COMMAND ${CMAKE_COMMAND} -E copy_directory
    #    "${CMAKE_SOURCE_DIR}/example/skins"
    #    "$<TARGET_FILE_DIR:${tgt_extended}>/skins")

     #add_custom_command(TARGET ${tgt_extended} POST_BUILD
     #   COMMAND ${CMAKE_COMMAND} -E copy_directory
     #   "${CMAKE_SOURCE_DIR}/example/icon"
     #   "$<TARGET_FILE_DIR:${tgt_extended}>/../icon")

     #add_custom_command(TARGET ${tgt_extended} POST_BUILD
     #   COMMAND ${CMAKE_COMMAND} -E copy_directory
     #   "${CMAKE_SOURCE_DIR}/example/images"
     #   "$<TARGET_FILE_DIR:${tgt_extended}>/../images")

     #add_custom_command(TARGET ${tgt_extended} POST_BUILD
     #   COMMAND ${CMAKE_COMMAND} -E copy_directory
     #   "${CMAKE_SOURCE_DIR}/extra_font"
     #   "$<TARGET_FILE_DIR:${tgt_extended}>/../../extra_font")

     #add_custom_command(TARGET ${tgt_skinning} POST_BUILD
     #   COMMAND ${CMAKE_COMMAND} -E copy_directory
     #   "${CMAKE_SOURCE_DIR}/example/skins"
     #   "$<TARGET_FILE_DIR:${tgt_skinning}>/../skins")

endfunction()


project ("nuklear_demos" LANGUAGES C)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
if(EMSCRIPTEN)
    set(CMAKE_C_EXTENSIONS ON)
else()
    set(CMAKE_C_EXTENSIONS OFF)
endif()

#nk_add_demo_sdl2_opengl3()

#nk_require_glew()
#nk_add_glfw_examples()
nk_add_demo_sdl3_renderer()
#nk_add_glfw_examples()

#pacman -S mingw-w64-clang-x86_64-emscripten
#emcmake cmake -S . -B build-ems -G Ninja
#cmake --build build-ems -j
