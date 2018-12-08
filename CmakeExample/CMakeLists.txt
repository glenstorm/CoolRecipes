cmake_minimum_required( VERSION 3.9 )
set(PROJECT_NAME rest_project )
project( ${PROJECT_NAME} )

# link dependencies
if (NOT TARGET some_library)
	add_subdirectory( ../some_library some_library )
endif()

#
if( UNIX )
add_custom_target(
	run ALL
	COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/../run_some_script.sh
	BYPRODUCTS ${CMAKE_CURRENT_SOURCE_DIR}/src/output.cpp ${CMAKE_CURRENT_SOURCE_DIR}/src/output.h
	COMMENT "som_comment"
)
endif()



set( CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -O0 -rdynamic" )
set( CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3 -rdynamic -s" )
set( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -O0 -rdynamic")
set( CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -rdynamic" )

if( UNIX )
	if( CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.9 )
		set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color")
		set( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -std=c++11")
		set( CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -std=c++11" )
	else( CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.9 )
		set( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -std=c++0x")
		set( CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -std=c++0x" )
	endif( CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.9 )
	set( CMAKE_SHARED_LINKER_FLAGS "-Wl,--no-undefined" )
endif( UNIX )

# sources to include in project
set( SOURCE
some_source.cpp
)

# reduce warnings for special files
set( GENTLE_COMPILE_SOURCE
some_source_with_warnings.cpp
)

set_source_files_properties( ${GENTLE_COMPILE_SOURCE} PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -Wno-sign-compare -Wno-uninitialized -Wno-unused-variable -Wno-unused-function" )

# reduce warnings while linking
if( UNIX )
	set( CMAKE_SHARED_LINKER_FLAGS "-Wl,--no-undefined" )
endif( UNIX )

set(LINK_FLAGS ${LINK_FLAGS} "-Wl,-whole-archive")

# include --version parameter
if(EXISTS "${CMAKE_SOURCE_DIR}/../.git")
	execute_process(
	COMMAND git rev-parse --abbrev-ref HEAD
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
	OUTPUT_VARIABLE GIT_BRANCH
	OUTPUT_STRIP_TRAILING_WHITESPACE
	)
else( EXISTS "${CMAKE_SOURCE_DIR}/../.git" )
	set(GIT_BRANCH "")
endif( EXISTS "${CMAKE_SOURCE_DIR}/../.git" )


string( TIMESTAMP VERSION_DATE_TIME "_%Y_%m_%d_%H_%M" )
string( CONCAT CONCAT_VERSION ${GIT_BRANCH} ${VERSION_DATE_TIME} )
set( GIT_BRANCH ${CONCAT_VERSION} )
message( STATUS "VERSION_DATE_TIME: ${CONCAT_VERSION}" )

# use GIT_BRANCH in cpp source to show application build information.
add_definitions( "-DGIT_BRANCH=${CONCAT_VERSION}" )
message( STATUS "Git current branch: ${CONCAT_VERSION}" )

configure_file(
	${CMAKE_SOURCE_DIR}/include/version.h.in
	${CMAKE_BINARY_DIR}/generated/version.h
)


include_directories(
${PROJECT_SOURCE_DIR}
${CMAKE_BINARY_DIR}/generated
${CMAKE_SOURCE_DIR}/include
)

#find libxml2
find_package(LibXml2 REQUIRED)
if(NOT LIBXML2_FOUND)
	message(SEND_ERROR "Failed to find LibXml2")
	return()
else()
	include_directories(${LIBXML2_INCLUDE_DIR})
endif()

#find Curses
find_package( Curses REQUIRED )
if(NOT Curses_FOUND)
	message(SEND_ERROR "Failed to find ncurses")
	return()
else()
	include_directories(${CURSES_INCLUDE_DIR})
endif()

#find curl
find_package( CURL REQUIRED )
if(NOT CURL_FOUND)
	message(SEND_ERROR "Failed to find CURL")
	return()
else()
	include_directories(${CURL_INCLUDE_DIRS})
endif()

#find boost
add_definitions( -DBOOST_FILESYSTEM_VERSION=3 )
add_definitions( -DBOOST_ASIO_DISABLE_STD_CHRONO )
add_definitions( -DBOOST_PARAMETER_MAX_ARITY=30 )

# # set( Boost_USE_STATIC_LIBS ON )
find_package( Boost COMPONENTS system filesystem thread program_options regex date_time chrono REQUIRED )
if(NOT Boost_FOUND)
	message(SEND_ERROR "Failed to find boost::{system|filesystem|thread|program_options|regex|date_time|chrono}")
	return()
else()
	include_directories(${Boost_INCLUDE_DIRS})
endif()

#turn on ccach if installed
find_program( CCACHE_FOUND ccache )
if( CCACHE_FOUND )
    set_property( GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache )
    set_property( GLOBAL PROPERTY RULE_LAUNCH_LINK ccache )
endif( CCACHE_FOUND )

##########################################################
add_executable( ${PROJECT_NAME} ${SOURCE} )

target_link_libraries( 	${PROJECT_NAME}
						${Boost_LIBRARIES}
						${CURL_LIBRARIES}
						${LIBXML2_LIBRARIES}
)

# pre-run script "run"
add_dependencies( ${PROJECT_NAME} run )
