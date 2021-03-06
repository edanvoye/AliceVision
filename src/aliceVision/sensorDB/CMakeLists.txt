add_definitions(-DTHIS_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}")

# Headers
set(sensorDB_files_headers
  Datasheet.hpp
  parseDatabase.hpp
)

# Sources
set(sensorDB_files_sources
  Datasheet.cpp
  parseDatabase.cpp
)

alicevision_add_library(aliceVision_sensorDB
  SOURCES ${sensorDB_files_headers} ${sensorDB_files_sources}
  PRIVATE_LINKS
    ${Boost_FILESYSTEM_LIBRARY}
    ${Boost_SYSTEM_LIBRARY}
  PRIVATE_INCLUDE_DIRS
    ${Boost_INCLUDE_DIRS}
)

# Install sensor_width_camera_database.txt
install(FILES sensor_width_camera_database.txt
  DESTINATION share/aliceVision
)

# Unit tests
alicevision_add_test(parseDatabase_test.cpp NAME "sensorDB_parseDatabase" LINKS aliceVision_sensorDB)



