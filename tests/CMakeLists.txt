project(tests C)

include_directories(
  ${CMAKE_BINARY_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/src
  ${CMOCKA_INCLUDE_DIR}
)

set(TORTURE_LIBRARY torture)

# A simple DNS server for testing
add_executable(dns_srv dns_srv.c)
target_link_libraries(dns_srv ${RWRAP_REQUIRED_LIBRARIES})

add_executable(test_real_res_query test_real_res_query.c)
target_link_libraries(test_real_res_query ${RWRAP_REQUIRED_LIBRARIES} ${CMOCKA_LIBRARY})

configure_file(fake_hosts.in ${CMAKE_CURRENT_BINARY_DIR}/fake_hosts @ONLY)

add_library(${TORTURE_LIBRARY} STATIC torture.c)
target_link_libraries(${TORTURE_LIBRARY}
    ${CMOCKA_LIBRARY}
    ${SWRAP_REQUIRED_LIBRARIES})


set(TESTSUITE_LIBRARIES ${RWRAP_REQUIRED_LIBRARIES} ${CMOCKA_LIBRARY})
if (HAVE_LIBRESOLV)
    set(TESTSUITE_LIBRARIES ${TESTSUITE_LIBRARIES} resolv)
endif()

set(RWRAP_TESTS
    test_res_init)

set(PRELOAD_LIBS ${RESOLV_WRAPPER_LOCATION})

# Some tests require socket_wrapper as well.
find_package(socket_wrapper)

# On Solaris the socket functions are compiled into libresolv.so so we can't preload
# socket_wrapper. Only faking will work!
if (HAVE_LIBRESOLV AND SOCKET_WRAPPER_LIBRARY AND NOT SOLARIS)
    set(RWRAP_TESTS ${RWRAP_TESTS} test_res_query_search)
    set(PRELOAD_LIBS ${RESOLV_WRAPPER_LOCATION}:${SOCKET_WRAPPER_LIBRARY})
endif()

foreach(_RWRAP_TEST ${RWRAP_TESTS})
    add_cmocka_test(${_RWRAP_TEST} ${_RWRAP_TEST}.c ${TORTURE_LIBRARY} ${TESTSUITE_LIBRARIES})

    if (OSX)
        set_property(
            TEST
                ${_RWRAP_TEST}
            PROPERTY
                ENVIRONMENT DYLD_FORCE_FLAT_NAMESPACE=1;DYLD_INSERT_LIBRARIES=${PRELOAD_LIBS})
    else ()
        set_property(
            TEST
                ${_RWRAP_TEST}
            PROPERTY
                ENVIRONMENT LD_PRELOAD=${PRELOAD_LIBS})
    endif()
endforeach()

add_cmocka_test(test_dns_fake test_dns_fake.c ${TORTURE_LIBRARY} ${TESTSUITE_LIBRARIES})
if (OSX)
    set_property(
        TEST
            test_dns_fake
        PROPERTY
        ENVIRONMENT DYLD_FORCE_FLAT_NAMESPACE=1;DYLD_INSERT_LIBRARIES=${PRELOAD_LIBS};RESOLV_WRAPPER_HOSTS=${CMAKE_CURRENT_BINARY_DIR}/fake_hosts)
else ()
    set_property(
        TEST
            test_dns_fake
        PROPERTY
            ENVIRONMENT LD_PRELOAD=${PRELOAD_LIBS};RESOLV_WRAPPER_HOSTS=${CMAKE_CURRENT_BINARY_DIR}/fake_hosts)
endif ()
