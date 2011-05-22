# $Id$

if (MAINTAINER)
  set (MANDIR ${PROJECT_BINARY_DIR}/man)
else (MAINTAINER)
  set (MANDIR ${PROJECT_SOURCE_DIR}/man)
endif (MAINTAINER)

include_directories (${MANDIR})

add_custom_target (tools ALL)
foreach (TOOL ${TOOLS})

  add_executable (${TOOL} ${TOOL}.cpp)
  if (MAINTAINER)
    add_dependencies (${TOOL} usage)
  endif (MAINTAINER)
  add_dependencies (tools ${TOOL})

  set_source_files_properties (${TOOL}.cpp PROPERTIES
    OBJECT_DEPENDS ${MANDIR}/${TOOL}.usage)

  target_link_libraries (${TOOL} Geographic)
# set_target_properties (${TOOL} PROPERTIES VERSION "${LIBVERSIONFULL}")
endforeach (TOOL)

install (TARGETS ${TOOLS} DESTINATION bin)

if (NOT WIN32)
  set (GEOID_SCRIPT geographiclib-get-geoids)
  add_custom_target (${GEOID_SCRIPT} ALL 
    COMMAND
      sed -e "s%@GEOID_DEFAULT_PATH@%${GEOGRAPHICLIB_GEOID_PATH}%"
	${CMAKE_CURRENT_SOURCE_DIR}/${GEOID_SCRIPT}.sh >
	${GEOID_SCRIPT}
    COMMAND chmod +x ${GEOID_SCRIPT}
    DEPENDS ${GEOID_SCRIPT}.sh)
  install (PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${GEOID_SCRIPT}
    DESTINATION sbin)
endif (NOT WIN32)

if (NOT WIN32 OR NOT GEOGRAPHIC_SHARED_LIB)
  # testing doesn't work for windows with shared libraries because the
  # dlls are not found
  enable_testing ()
endif (NOT WIN32 OR NOT GEOGRAPHIC_SHARED_LIB)

add_test (GeoConvert0 GeoConvert -p -3 -m --input-string "33.3 44.4")
set_tests_properties (GeoConvert0
  PROPERTIES PASS_REGULAR_EXPRESSION "38SMB4484")
add_test (GeoConvert1 GeoConvert -d --input-string "38smb")
set_tests_properties (GeoConvert1
  PROPERTIES PASS_REGULAR_EXPRESSION "32d59'14\\.1\"N 044d27'53\\.4\"E")
add_test (GeoConvert2 GeoConvert -p -2 --input-string "30d30'30\" 30.50833")
set_tests_properties (GeoConvert2
  PROPERTIES PASS_REGULAR_EXPRESSION "30\\.508 30\\.508")
add_test (GeoConvert3 GeoConvert --junk)
set_tests_properties (GeoConvert3 PROPERTIES WILL_FAIL ON)
add_test (GeoConvert4 GeoConvert --input-string garbage)
set_tests_properties (GeoConvert4 PROPERTIES WILL_FAIL ON)

add_test (Geod0 Geod -i -p 0 --input-string "40.6 -73.8 49d01'N 2d33'E")
set_tests_properties (Geod0
  PROPERTIES PASS_REGULAR_EXPRESSION "53\\.47022 111\\.59367 5853226 5066469")
add_test (Geod1
  Geod -p 0 --input-string "40d38'23\"N 073d46'44\"W 53d30' 5850e3")
set_tests_properties (Geod1
  PROPERTIES PASS_REGULAR_EXPRESSION "49\\.01467 2\\.56106 111\\.62947 5064509")
# Check fix for antipodal prolate bug found 2010-09-04
add_test (Geod2
  Geod -i -p 0 -e 6.4e6 -150 --input-string "0.07476 0 -0.07476 180")
set_tests_properties (Geod2
  PROPERTIES PASS_REGULAR_EXPRESSION "90\\.00078 90\\.00078 20106193 134031")
# Another check for similar bug
add_test (Geod3
  Geod -i -p 0 -e 6.4e6 -150 --input-string "0.1 0 -0.1 180")
set_tests_properties (Geod3
  PROPERTIES PASS_REGULAR_EXPRESSION "90\\.00105 90\\.00105 20106193 134031")
# Check fix for short line bug found 2010-05-21
add_test (Geod4
  Geod -i --input-string "36.493349428792 0 36.49334942879201 .0000008")
set_tests_properties (Geod4
  PROPERTIES PASS_REGULAR_EXPRESSION
  "89\\.99999948 89\\.99999996 0\\.072 0\\.072")
# Check fix for point2=pole bug found 2010-05-03 (but only with long double)
add_test (Geod5
  Geod -p 0 --input-string "0.01777745589997 30 0 10e6")
set_tests_properties (Geod5
  PROPERTIES PASS_REGULAR_EXPRESSION
  "90\\.00000 -150\\.00000 -180\\.00000 6378137;90\\.00000 30\\.00000 0\\.00000 6378137")

# Check fix for pole-encircling bug found 2011-03-16
add_test (Planimeter0 Planimeter --input-string "89 0;89 90;89 180;89 270")
add_test (Planimeter1 Planimeter --input-string "0 -1;-1 0;0 1;1 0")
add_test (Planimeter2
  Planimeter -r --input-string "-89 0;-89 90;-89 180;-89 270")
set_tests_properties (Planimeter0
  PROPERTIES PASS_REGULAR_EXPRESSION
  "4 631819\\.8745[0-9]+ 2495230567[78]\\.[0-9]+")
set_tests_properties (Planimeter2
  PROPERTIES PASS_REGULAR_EXPRESSION
  "4 631819\\.8745[0-9]+ 2495230567[78]\\.[0-9]+")
set_tests_properties (Planimeter1
  PROPERTIES PASS_REGULAR_EXPRESSION "4 627598\\.2731[0-9]+ 24619419146.[0-9]+")

# Check fix for AlbersEqualArea::Reverse bug found 2011-05-01
add_test (ConicProj0
  ConicProj -a 40d58 39d56 -l 77d45W -r --input-string "220e3 -52e3")
set_tests_properties (ConicProj0
  PROPERTIES PASS_REGULAR_EXPRESSION
  "39.95[0-9]+ -75.17[0-9]+ 1.67[0-9]+ 0.99[0-9]+")


if (EXISTS ${GEOGRAPHICLIB_GEOID_PATH}/egm96-5.pgm)
  # Check fix for single-cell cache bug found 2010-11-23
  add_test (GeoidEval0 GeoidEval --input-string "0d1 0d1;0d4 0d4")
  set_tests_properties (GeoidEval0
    PROPERTIES PASS_REGULAR_EXPRESSION "^17\\.1602 .*17\\.1511 " )
endif (EXISTS ${GEOGRAPHICLIB_GEOID_PATH}/egm96-5.pgm)