
cmake_minimum_required(VERSION 3.5)

project(cpp3k CXX)

include_directories(include)


install(DIRECTORY include/cpp3k DESTINATION include)


# Don't build examples as part of the main build.
# add_subdirectory(examples)
