set(timeout 540)  # The timeout in seconds for individual tests.
set(CTEST_OUTPUT_ON_FAILURE TRUE)

include(GoogleTest)

hunter_add_package(GTest)
find_package(GTest CONFIG REQUIRED)

set(unittest_sources
    unittests/libdevcore/CommonJS.cpp
    unittests/libdevcore/core.cpp
    unittests/libdevcore/FixedHash.cpp
    unittests/libdevcore/RangeMask.cpp
    unittests/libdevcore/RLP.cpp

    unittests/libdevcrypto/AES.cpp

    unittests/libethashseal/EthashTest.cpp

    unittests/libethcore/BlockHeader.cpp
    unittests/libethcore/CommonJS.cpp
    unittests/libethcore/KeyManager.cpp

    unittests/libethereum/ExecutiveTest.cpp
    unittests/libethereum/ValidationSchemes.cpp

    unittests/libp2p/capability.cpp
    unittests/libp2p/eip-8.cpp
    unittests/libp2p/EndpointTrackerTest.cpp
    unittests/libp2p/ENRTest.cpp
    unittests/libp2p/rlpx.cpp

    unittests/libweb3core/memorydb.cpp
    unittests/libweb3core/overlaydb.cpp
    unittests/libweb3core/statecachedb.cpp

    unittests/libweb3jsonrpc/AccountHolder.cpp
)

add_executable(aleth-unittests ${unittest_sources})
target_include_directories(aleth-unittests PRIVATE ${UTILS_INCLUDE_DIR})
target_link_libraries(aleth-unittests PRIVATE
    web3jsonrpc ethashseal devcrypto devcore
    GTest::gtest GTest::main
)
gtest_add_tests(TARGET aleth-unittests TEST_PREFIX unittests/ TEST_LIST unittests)
set_tests_properties(${unittests} PROPERTIES TIMEOUT ${timeout})


file(GLOB_RECURSE sources RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "*.cpp" "*.h" "*.sol" )

# Skip unit tests included in aleth-unittests.
list(REMOVE_ITEM sources ${unittest_sources})

# search for test names and create ctest tests
set(excludeSuites jsonrpc \"customTestSuite\" BlockQueueSuite)
set(allSuites jsonrpc)
set(allTests "")
foreach(file ${sources})
    file(STRINGS ${file} test_list_raw REGEX "BOOST_.*TEST_(SUITE|CASE|SUITE_END)")
    set(TestSuite "DEFAULT")
    set(TestSuitePath "")
    foreach(test_raw ${test_list_raw})
        string(REGEX REPLACE ".*TEST_(SUITE|CASE)\\(([^ ,\\)]*).*" "\\1 \\2" test ${test_raw})

        if(test MATCHES "^SUITE .*")
            string(SUBSTRING ${test} 6 -1 TestSuite)
            if (";${excludeSuites};" MATCHES ";${TestSuite};")
                continue()
            endif()

            set(TestSuitePath "${TestSuitePath}/${TestSuite}")
            string(SUBSTRING ${TestSuitePath} 1 -1 TestSuitePathFixed)
            if(FASTCTEST)
                if (NOT ";${allSuites};" MATCHES ";${TestSuite};")
                    list(APPEND allSuites ${TestSuite})
                    separate_arguments(TESTETH_ARGS)
                    set(TestEthArgs -t ${TestSuitePathFixed} -- ${TESTETH_ARGS})
                    add_test(NAME ${TestSuitePathFixed} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/test COMMAND testeth ${TestEthArgs})
                    set_tests_properties(${TestSuitePathFixed} PROPERTIES TIMEOUT ${timeout})
                endif()
            endif()
            set(allTests "${allTests}   \"${TestSuitePathFixed}\",\n")
        elseif(test MATCHES "^CASE .*")
            #skip all test cases for the testSuite that is being skipped
            if (";${excludeSuites};" MATCHES ";${TestSuite};")
                continue()
            endif()
            if(NOT FASTCTEST)
                if(NOT test MATCHES "^CASE &createRandom.*")
                    string(SUBSTRING ${test} 5 -1 TestCase)
                    string(SUBSTRING ${TestSuitePath} 1 -1 TestSuitePathFixed)
                    separate_arguments(TESTETH_ARGS)
                    set(TestEthArgs -t ${TestSuitePathFixed}/${TestCase} -- ${TESTETH_ARGS})
                    add_test(NAME ${TestSuitePathFixed}/${TestCase} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/test COMMAND testeth ${TestEthArgs})
                    set_tests_properties(${TestSuitePathFixed}/${TestCase} PROPERTIES TIMEOUT ${timeout})
                    set(allTests "${allTests}   \"${TestSuitePathFixed}/${TestCase}\",\n")
                endif()
            endif()
        elseif (";${test_raw};" MATCHES "BOOST_AUTO_TEST_SUITE_END()")
            #skip all suite end for the testSuite that is being skipped
            if (";${excludeSuites};" MATCHES ";${TestSuite};")
                continue()
            endif()
            #encountered SUITE_END block. remove one test suite from the suite path.
            string(FIND ${TestSuitePath} "/" Position REVERSE)
            string(SUBSTRING ${TestSuitePath} 0 ${Position} TestSuitePath)
        endif()
    endforeach(test_raw)
endforeach(file)

#Save allTests into a variable in include file
configure_file(tools/libtesteth/AllTestNames.h.in ${PROJECT_BINARY_DIR}/AllTestNames.h)

hunter_add_package(yaml-cpp)
find_package(yaml-cpp CONFIG REQUIRED)

add_executable(testeth ${sources})
target_include_directories(testeth PRIVATE ${UTILS_INCLUDE_DIR})
target_link_libraries(testeth PRIVATE ethereum ethashseal web3jsonrpc devcrypto devcore aleth-buildinfo cryptopp-static yaml-cpp::yaml-cpp binaryen::binaryen libjson-rpc-cpp::client)
install(TARGETS testeth DESTINATION ${CMAKE_INSTALL_BINDIR})

set_property(SOURCE tools/libtesteth/TestHelper.cpp PROPERTY COMPILE_DEFINITIONS BINARYEN_VERSION=${BINARYEN_VERSION})

include(EthUtils)

eth_add_test(ClientBase
    ARGS --eth_testfile=BlockTests/bcJS_API_Test --eth_threads=1
    ARGS --eth_testfile=BlockTests/bcJS_API_Test --eth_threads=3
    ARGS --eth_testfile=BlockTests/bcJS_API_Test --eth_threads=10
    ARGS --eth_testfile=BlockTests/bcValidBlockTest --eth_threads=1
    ARGS --eth_testfile=BlockTests/bcValidBlockTest --eth_threads=3
    ARGS --eth_testfile=BlockTests/bcValidBlockTest --eth_threads=10
)

#Does not work
#eth_add_test(JsonRpc
#   ARGS --eth_testfile=BlockTests/bcJS_API_Test
#   ARGS --eth_testfile=BlockTests/bcValidBlockTest
#)

