cmake_minimum_required(VERSION 2.8.4)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules")
project(SOEM C)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
   # Default to installing in SOEM source directory
   set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install)
endif()

set(SOEM_INCLUDE_INSTALL_DIR include/soem)
set(SOEM_LIB_INSTALL_DIR lib)
set(BUILD_TESTS TRUE)

option( XENOMAI_COBALT_KERNEL "Enable the XENOMAI COBALT KERNEL" ON )

#######
# WIN32
#######
if(WIN32)
  set(OS "win32")
  include_directories(oshw/win32/wpcap/Include)
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    link_directories(${CMAKE_SOURCE_DIR}/oshw/win32/wpcap/Lib/x64)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    link_directories(${CMAKE_SOURCE_DIR}/oshw/win32/wpcap/Lib)
  endif()
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D _CRT_SECURE_NO_WARNINGS")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  /WX")
  set(OS_LIBS wpcap.lib Packet.lib Ws2_32.lib Winmm.lib)
#######
# WIN32 - END
#######

#######
# LINUX
#######
elseif(UNIX AND NOT APPLE)
  set(OS "linux")
  if( XENOMAI_COBALT_KERNEL )
    message("XENOMAI_COBALT_KERNEL is REQUIRED")
    find_package(Xenomai REQUIRED )
    
    include_directories( ${XENOMAI_POSIX_INCLUDE_DIRS}        )
    add_definitions    ( ${XENOMAI_POSIX_COMPILE_DEFINITIONS} )
    
    set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} ${XENOMAI_POSIX_CFLAGS} )
    set(OS_LIBS       ${XENOMAI_POSIX_LIBRARIES} )
    set(OS_LDFLAGS    ${XENOMAI_POSIX_LDFLAGS}  )
    
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
    set(OS_LIBS pthread rt)
  endif()
  
#######
# LINUX - END
#######


#######
# APPLE
#######
elseif(APPLE)
  # This must come *before* linux or MacOSX will identify as Unix.
  set(OS "macosx")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
  set(OS_LIBS pthread pcap)
#######
# APPLE - END
#######
  
#######
# RT-KERNEL
#######
elseif(${CMAKE_SYSTEM_NAME} MATCHES "rt-kernel")
  set(OS "rtk")
  message("ARCH is ${ARCH}")
  message("BSP is ${BSP}")
  include_directories(oshw/${OS}/${ARCH})
  file(GLOB OSHW_EXTRA_SOURCES oshw/${OS}/${ARCH}/*.c)
  set(OSHW_SOURCES "${OS_HW_SOURCES} ${OSHW_ARCHSOURCES}")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-but-set-variable")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-function")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format")
  set(OS_LIBS "-Wl,--start-group -l${BSP} -l${ARCH} -lkern -ldev -lsio -lblock -lfs -lusb -llwip -leth -li2c -lrtc -lcan -lnand -lspi -lnor -lpwm -ladc -ltrace -lc -lm -Wl,--end-group")

#######
# RT-KERNEL
#######
elseif(${CMAKE_SYSTEM_NAME} MATCHES "rtems")
  message("Building for RTEMS")
  set(OS "rtems")
  set(SOEM_LIB_INSTALL_DIR ${LIB_DIR})
  set(BUILD_TESTS FALSE)
  
#######
# RT-KERNEL - END
#######
endif()

message("OS is ${OS}")

file(GLOB SOEM_SOURCES soem/*.c)
file(GLOB OSAL_SOURCES osal/${OS}/*.c)
file(GLOB OSHW_SOURCES oshw/${OS}/*.c)

file(GLOB SOEM_HEADERS soem/*.h)
file(GLOB OSAL_HEADERS osal/osal.h osal/${OS}/*.h)
file(GLOB OSHW_HEADERS oshw/${OS}/*.h)

include_directories(soem)
include_directories(osal)
include_directories(osal/${OS})
include_directories(oshw/${OS})

add_library(soem STATIC
  ${SOEM_SOURCES}
  ${OSAL_SOURCES}
  ${OSHW_SOURCES}
  ${OSHW_EXTRA_SOURCES})
target_link_libraries(soem ${OS_LIBS} )
set_target_properties(soem PROPERTIES LINK_FLAGS ${OS_LDFLAGS} )

message("LIB_DIR: ${SOEM_LIB_INSTALL_DIR}")

install(TARGETS soem DESTINATION ${SOEM_LIB_INSTALL_DIR})
install(FILES
  ${SOEM_HEADERS}
  ${OSAL_HEADERS}
  ${OSHW_HEADERS}
  DESTINATION ${SOEM_INCLUDE_INSTALL_DIR})

if(BUILD_TESTS) 
  add_subdirectory(test/linux/slaveinfo)
  add_subdirectory(test/linux/eepromtool)
  add_subdirectory(test/linux/simple_test)
endif()
