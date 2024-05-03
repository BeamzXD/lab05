cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TEST "Build tests" OFF)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

if(BUILD_TESTS)
	enable_testing()
	add_subdirectory(third-party/gtest)
	add_executable(check tests/tests.cpp)
	target_link_libraries(check banking gtest_main gmock_main)
	add_test(NAME check COMMAND check)
endif()