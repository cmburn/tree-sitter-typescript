cmake_minimum_required(VERSION 3.13)

project(tree-sitter-typescript
        VERSION "0.23.2"
		DESCRIPTION "TypeScript and TSX grammars for tree-sitter"
        HOMEPAGE_URL "https://github.com/tree-sitter/tree-sitter-typescript"
        LANGUAGES C)

option(BUILD_SHARED_LIBS "Build using shared libraries" ON)
option(TREE_SITTER_REUSE_ALLOCATOR "Reuse the library allocator" OFF)

set(TREE_SITTER_ABI_VERSION 14 CACHE STRING "Tree-sitter ABI version")
if(NOT ${TREE_SITTER_ABI_VERSION} MATCHES "^[0-9]+$")
    unset(TREE_SITTER_ABI_VERSION CACHE)
    message(FATAL_ERROR "TREE_SITTER_ABI_VERSION must be an integer")
endif()

find_program(TREE_SITTER_CLI tree-sitter DOC "Tree-sitter CLI")

include(GNUInstallDirs)

set(TREE_SITTER_TYPESCRIPT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

macro(add_parser name)
    add_custom_command(OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/src/parser.c"
                       DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/grammar.json"
                       COMMAND "${TREE_SITTER_CLI}" generate src/grammar.json
                                --abi=${TREE_SITTER_ABI_VERSION}
                       WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
                       COMMENT "Generating parser.c")

    add_library(tree-sitter-${name}
                "${CMAKE_CURRENT_SOURCE_DIR}/src/parser.c"
                "${CMAKE_CURRENT_SOURCE_DIR}/src/scanner.c")
    target_include_directories(tree-sitter-${name} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/src")

    target_compile_definitions(tree-sitter-${name} PRIVATE
                               $<$<BOOL:${TREE_SITTER_REUSE_ALLOCATOR}>:TREE_SITTER_REUSE_ALLOCATOR>
                               $<$<CONFIG:Debug>:TREE_SITTER_DEBUG>)

    set_target_properties(tree-sitter-${name}
                          PROPERTIES
                          C_STANDARD 11
                          POSITION_INDEPENDENT_CODE ON
                          SOVERSION "${TREE_SITTER_ABI_VERSION}.${PROJECT_VERSION_MAJOR}"
                          DEFINE_SYMBOL "")

    configure_file("${TREE_SITTER_TYPESCRIPT_SOURCE_DIR}/bindings/c/tree-sitter-${name}.pc.in"
                   "${CMAKE_CURRENT_BINARY_DIR}/tree-sitter-${name}.pc" @ONLY)

    install(FILES "${TREE_SITTER_TYPESCRIPT_SOURCE_DIR}/bindings/c/tree-sitter-${name}.h"
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/tree_sitter")
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/tree-sitter-${name}.pc"
            DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/pkgconfig")
    install(TARGETS tree-sitter-${name}
            LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
endmacro()

add_subdirectory(typescript tree-sitter-typescript)

add_subdirectory(tsx tree-sitter-tsx)

add_custom_target(ts-test "${TREE_SITTER_CLI}" test
                  WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
                  COMMENT "tree-sitter test")
