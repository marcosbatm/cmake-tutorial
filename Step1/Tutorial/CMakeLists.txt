# TODO10: Move all the Tutorial target commands to this CMakeLists.txt. Ensure
#          that all paths are updated to be relative to this new location.

# TODO3: Add an executable target called Tutorial to the project
add_executable(Tutorial)

# TODO7: Add the MathFunctions library as a linked dependency
#        to the Tutorial target
target_link_libraries(Tutorial
  PRIVATE
    MathFunctions
)

# TODO4: Add the Tutorial/Tutorial.cxx source file to the Tutorial target
target_sources(Tutorial 
  PRIVATE 
    Tutorial.cxx
)