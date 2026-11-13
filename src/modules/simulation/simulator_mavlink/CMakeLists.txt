############################################################################
#
#   Copyright (c) 2015-2022 PX4 Development Team. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in
#    the documentation and/or other materials provided with the
#    distribution.
# 3. Neither the name PX4 nor the names of its contributors may be
#    used to endorse or promote products derived from this software
#    without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
# FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
# COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
# INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
# OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
# AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#
############################################################################

px4_add_module(
	MODULE modules__simulation__simulator_mavlink
	MAIN simulator_mavlink
	COMPILE_FLAGS
		-Wno-double-promotion
		-Wno-cast-align
		-Wno-address-of-packed-member # TODO: fix in c_library_v2
	INCLUDES
		${CMAKE_BINARY_DIR}/mavlink
		${CMAKE_BINARY_DIR}/mavlink/${CONFIG_MAVLINK_DIALECT}
	SRCS
		SimulatorMavlink.cpp
		SimulatorMavlink.hpp
	DEPENDS
		mavlink_c_generate
		conversion
		geo
		drivers_accelerometer
		drivers_gyroscope
		drivers_magnetometer
	)

# Optional GeographicLib support for geoid height (EGM96) in HIL_GPS handling.
# GeographicLib is taken from commonlibs, located next to the PX4 source tree:
#   ${PX4_SOURCE_DIR}/../commonlibs/lib/libGeographicLib.a   (built by commonlibs/build.sh)
#   ${PX4_SOURCE_DIR}/../commonlibs/include/GeographicLib/
#   ${PX4_SOURCE_DIR}/../commonlibs/geoids/egm96-5.pgm      (geoid grid, runtime)
# Only GeographicLib is used; libCommonLibs.a (GeoMath, C++20) is not linked.
# If GeographicLib is not found, the module builds without geoid correction (N = 0).
get_filename_component(SIM_COMMONLIBS_ROOT "${PX4_SOURCE_DIR}/../commonlibs" ABSOLUTE)
set(SIM_GEOGRAPHICLIB_LIB "${SIM_COMMONLIBS_ROOT}/lib/libGeographicLib.a")
set(SIM_GEOGRAPHICLIB_INC "${SIM_COMMONLIBS_ROOT}/include")
set(SIM_GEOID_DATA_DIR "${SIM_COMMONLIBS_ROOT}/geoids")

if(EXISTS "${SIM_GEOGRAPHICLIB_LIB}" AND EXISTS "${SIM_GEOGRAPHICLIB_INC}/GeographicLib/Geoid.hpp")
	message(STATUS "simulator_mavlink: GeographicLib found in ${SIM_COMMONLIBS_ROOT} (geoid correction enabled)")
	# SYSTEM include so third-party header warnings do not break the PX4 build
	target_include_directories(modules__simulation__simulator_mavlink SYSTEM PRIVATE "${SIM_GEOGRAPHICLIB_INC}")
	target_link_libraries(modules__simulation__simulator_mavlink PRIVATE "${SIM_GEOGRAPHICLIB_LIB}")
	target_compile_definitions(modules__simulation__simulator_mavlink PRIVATE
		PX4_SIM_GEOID_ENABLED=1
		PX4_SIM_GEOID_DATA_DIR="${SIM_GEOID_DATA_DIR}"
	)
else()
	message(WARNING "simulator_mavlink: GeographicLib not found in ${SIM_COMMONLIBS_ROOT}, building without geoid correction (N = 0)")
endif()

include(sitl_targets_flightgear.cmake)
include(sitl_targets_gazebo-classic.cmake)
include(sitl_targets_jmavsim.cmake)
include(sitl_targets_jsbsim.cmake)


# none_iris (legacy compatibility launch helper)
add_custom_target(none_iris
	COMMAND ${CMAKE_COMMAND} -E env PX4_SYS_AUTOSTART=10016 $<TARGET_FILE:px4>
	WORKING_DIRECTORY ${SITL_WORKING_DIR}
	USES_TERMINAL
	DEPENDS
		px4
		${PX4_SOURCE_DIR}/ROMFS/px4fmu_common/init.d-posix/airframes/10016_none_iris
	COMMENT "launching px4 none_iris (SYS_AUTOSTART=10016)"
)
