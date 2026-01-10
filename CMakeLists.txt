cmake_minimum_required(VERSION 3.15)

project(
    poc-code-coverage-c
    VERSION 1.0.0
    DESCRIPTION "Proof of Concept Code Coverage in C"
    LANGUAGES C
)

include(CTest)
include(GNUInstallDirs)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(ENABLE_COVERAGE "Enable code coverage reporting" OFF)

# ------------------------------------------------------------------
# Library
# ------------------------------------------------------------------
file(GLOB_RECURSE PROJECT_SOURCES CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.c"
)

add_library(${PROJECT_NAME} ${PROJECT_SOURCES})
target_include_directories(${PROJECT_NAME}
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# ------------------------------------------------------------------
# Coverage flags (compiler detection)
# ------------------------------------------------------------------
if(ENABLE_COVERAGE)
    message(STATUS "${PROJECT_NAME}: Code coverage enabled")

    if(CMAKE_C_COMPILER_ID MATCHES "AppleClang|Clang")
        message(STATUS "${PROJECT_NAME}: Using LLVM coverage")

        set(COVERAGE_COMPILE_FLAGS
            -fprofile-instr-generate
            -fcoverage-mapping
        )
        set(COVERAGE_LINK_FLAGS
            -fprofile-instr-generate
        )
        set(COVERAGE_TYPE "llvm")

    elseif(CMAKE_C_COMPILER_ID STREQUAL "GNU")
        message(STATUS "${PROJECT_NAME}: Using gcov/lcov coverage")

        set(COVERAGE_COMPILE_FLAGS --coverage)
        set(COVERAGE_LINK_FLAGS --coverage)

        set(COVERAGE_TYPE "gcov")
    else()
        message(FATAL_ERROR "${PROJECT_NAME}: Coverage not supported for compiler: ${CMAKE_C_COMPILER_ID}")
    endif()

    target_compile_options(${PROJECT_NAME} PRIVATE ${COVERAGE_COMPILE_FLAGS})
    target_link_options(${PROJECT_NAME} PRIVATE ${COVERAGE_LINK_FLAGS})
endif()

# ------------------------------------------------------------------
# Tests
# ------------------------------------------------------------------
if(BUILD_TESTING)
    file(GLOB_RECURSE TEST_SOURCES CONFIGURE_DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/*.c"
    )

    foreach(test_source IN LISTS TEST_SOURCES)
        get_filename_component(test_name ${test_source} NAME_WE)

        add_executable(${test_name} ${test_source})
        target_link_libraries(${test_name} PRIVATE ${PROJECT_NAME})

        if(ENABLE_COVERAGE)
            target_compile_options(${test_name} PRIVATE ${COVERAGE_COMPILE_FLAGS})
            target_link_options(${test_name} PRIVATE ${COVERAGE_LINK_FLAGS})
        endif()

        add_test(NAME ${test_name} COMMAND ${test_name})

        message(STATUS "${PROJECT_NAME}: Added test ${test_name}")
    endforeach()
endif()

# ------------------------------------------------------------------
# Coverage target
# ------------------------------------------------------------------
if(ENABLE_COVERAGE)
    if(COVERAGE_TYPE STREQUAL "gcov")
        find_program(LCOV_PATH lcov REQUIRED)
        find_program(GENHTML_PATH genhtml REQUIRED)

        add_custom_target(coverage
            COMMAND ${LCOV_PATH} --capture --directory . --output-file coverage.info
            # Add any exclusion patterns here
            # COMMAND ${LCOV_PATH} --remove coverage.info '/usr/*' --output-file coverage.info
            COMMAND ${GENHTML_PATH} -o coverage_report coverage.info
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
            COMMENT "Generating gcov/lcov coverage report"
        )

    elseif(COVERAGE_TYPE STREQUAL "llvm")
        find_program(LLVM_PROFDATA llvm-profdata REQUIRED)
        find_program(LLVM_COV llvm-cov REQUIRED)

        add_custom_target(coverage
            COMMAND ${LLVM_PROFDATA} merge -sparse *.profraw -o coverage.profdata
            COMMAND ${LLVM_COV} export 
                $<TARGET_FILE:${PROJECT_NAME}> 
                -instr-profile=coverage.profdata 
                -format=lcov > coverage.info
            COMMAND ${LLVM_COV} show
                $<TARGET_FILE:${PROJECT_NAME}>
                -instr-profile=coverage.profdata
                -format=html
                -output-dir=coverage_report
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
            COMMENT "Generating LLVM coverage report"
        )
    endif()
endif()

# ------------------------------------------------------------------
# Install
# ------------------------------------------------------------------
install(TARGETS ${PROJECT_NAME})

install(
    DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
