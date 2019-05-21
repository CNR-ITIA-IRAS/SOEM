
set(SOURCES simple_test.c)
add_executable(simple_test ${SOURCES})

if (Xenomai_FOUND)
  target_link_libraries(simple_test soem ${XENOMAI_POSIX_LIBRARIES})
  set_target_properties(simple_test PROPERTIES LINK_FLAGS ${OS_LDFLAGS} )
elseif(WIN32)
   target_link_libraries(simple_test wpcap.lib Packet.lib Ws2_32.lib Winmm.lib)
elseif(UNIX)
   target_link_libraries(simple_test pthread rt)
endif()

install(TARGETS simple_test DESTINATION bin)
