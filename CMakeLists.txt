cmake_minimum_required (VERSION 2.6)
project (ZMAP C)
SET (VERSION 1.1.1)

option(WITH_REDIS "Build with support for Redis DB" OFF)
option(WITH_JSON "Build with support for JSON" OFF)
option(ENABLE_DEVELOPMENT "Enable development specific compiler and linker flags" OFF)
option(RESPECT_INSTALL_PREFIX_CONFIG "Respect CMAKE_INSTALL_PREFIX for /etc" OFF)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
	set(USING_CLANG "YES")
else()
	set(USING_GCC "YES")
endif()

# Hardening and warnings for building with gcc
# Maybe add -Wstrict-prototypes -Wmissing-prototypes -Wmissing-declarations
set(GCCWARNINGS 
	"-Wall -Wformat=2 -Wno-format-nonliteral"
	"-pedantic -fno-strict-aliasing"
	"-Wextra"
	"-Wfloat-equal -Wundef -Wwrite-strings -Wredundant-decls"
	"-Wnested-externs -Wbad-function-cast -Winit-self"
	"-Wmissing-noreturn"
	"-Wstack-protector"
	)

# Fix line breaks
string(REPLACE ";" " "  GCCWARNINGS "${GCCWARNINGS}")

if(ENABLE_DEVELOPMENT)
	set(GCCWARNINGS "${GCCWARNINGS} -Werror")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -g")
else()
	# Hardening and optimizations for building with gcc
	set(GCCHARDENING "-U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=2 -fstack-protector-all -fwrapv -fPIC --param ssp-buffer-size=1")
	if (NOT APPLE)
		set(LDHARDENING "-z relro -z now")
	else()
		set(LDHARDENING "")
	endif()

	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${GCCHARDENING} -O2")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LDHARDENING}")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${GCCWARNINGS}")

if(WITH_REDIS)
	set(REDIS_LIBS hiredis)
	add_definitions("-DREDIS")
endif()

if(WITH_JSON)
	include(FindPkgConfig)
	pkg_check_modules(JSON json-c)
	if(JSON_FOUND)
		include_directories(JSON_INCLUDE_DIRS)
	else()
		message(FATAL_ERROR "Did not find libjson")
	endif()

	add_definitions("-DJSON")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${JSON_CFLAGS}")
endif()



# Standard FLAGS
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu99")
if (NOT APPLE)
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pthread")
endif()

add_subdirectory(src)

# Install conf files
#FILE(GLOB CONF_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/conf" "conf/*")
#message(STATUS "${CONF_FILES}")
#configure_file(zmap_conf_install.cmake.in zmap_conf_install.cmake)
#install(SCRIPT "${PROJECT_SOURCE_DIR}/zmap_conf_install.cmake")

FILE(GLOB CONF_FILES "conf/*")
if (RESPECT_INSTALL_PREFIX_CONFIG)
	set(CONFIG_DESTINATION "etc/zmap")
else()
	set(CONFIG_DESTINATION "/etc/zmap")
endif()

install(
	FILES
	${CONF_FILES}
	DESTINATION ${CONFIG_DESTINATION}
)

# Allow Debian Packaging
INCLUDE (InstallRequiredSystemLibraries)

SET (CPACK_SET_DESTDIR "on")
SET (CPACK_PACKAGING_INSTALL_PREFIX "/tmp")
SET (CPACK_GENERATOR "DEB")

SET (${VERSION} CPACK_DEBIAN_PACKAGE_VERSION)
SET (CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
SET (CPACK_DEBIAN_PACKAGE_SECTION "network")
SET (CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})
SET (CPACK_DEBIAN_PACKAGE_DEPENDS "libc6 (>= 2.1.3), libgmp3c2, libpcap0.8")

SET (CPACK_PACKAGE_DESCRIPTION "Internet-scale network scanner")
SET (CPACK_PACKAGE_DESCRIPTION_SUMMARY "ZMap is an open-source network scanner that enables researchers to easily perform Internet-wide network studies. With a single machine and a well provisioned network uplink, ZMap is capable of performing a complete scan of the IPv4 address space in under 45 minutes, approaching the theoretical limit of gigabit Ethernet. ZMap can be used to study protocol adoption over time, monitor service availability, and help us better understand large systems distributed across the Internet.")
SET (CPACK_PACKAGE_CONTACT "Zakir Durumeric <zakird@gmail.com>")
SET (CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}_${VERSION}_${CPACK_DEBIAN_ARCHITECTURE}")

SET (CPACK_COMPONENTS_ALL Libraries ApplicationData)

INCLUDE(CPack)

