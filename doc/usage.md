
# Cmockery Unit Testing Framework
Cmockery is a lightweight library that is used to author C unit tests.

## Motivation
There are a variety of C unit testing frameworks available however many of them are fairly complex and require the latest compiler technology. Some development requires the use of old compilers which makes it difficult to use some unit testing frameworks. In addition many unit testing frameworks assume the code being tested is an application or module that is targeted to the same platform that will ultimately execute the test. Because of this assumption many frameworks require the inclusion of standard C library headers in the code module being tested which may collide with the custom or incomplete implementation of the C library utilized by the code under test.

Cmockery only requires a test application is linked with the standard C library which minimizes conflicts with standard C library headers. Also, Cmockery tries avoid the use of some of the newer features of C compilers.

This results in Cmockery being a relatively small library that can be used to test a variety of exotic code. If a developer wishes to simply test an application with the latest compiler then other unit testing frameworks maybe preferable.

## Overview
Cmockery tests are compiled into stand-alone executables and linked with the Cmockery library, the standard C library and module being tested. Any symbols external to the module being tested should be mocked - replaced with functions that return values determined by the test - within the test application. Even though significant differences may exist between the target execution environment of a code module and the environment used to test the code the unit testing is still valid since its goal is to test the logic of a code modules at a functional level and not necessarily all of its interactions with the target execution environment.

It may not be possible to compile a module into a test application without some modification, therefore the preprocessor symbol UNIT_TESTING should be defined when Cmockery unit test applications are compiled so code within the module can be conditionally compiled for tests.

## Installation

### Binary Installation
Cmockery can be installed on Fedora/EPEL systems by doing the following:

```
$ sudo yum install cmockery2 cmockery2-devel
```

### Installing from the source

#### Environment Setup

* Ubuntu

```sh
$ sudo apt-get -y install libtool make autoconf automake gcc
```

* Fedora/RHEL

```sh
$ sudo yum -y install libtool make autoconf automake gcc
```

* Mac (with Homebrew)
```sh
xcode-select --install # if we to install clang

$ brew install autoconf automake libtool
```

#### Build cmockery

```sh
$ ./autogen.sh
$ ./configure
$ make
```

## Test Execution
Cmockery unit test cases are functions with the signature void `function(void **state)`. Cmockery test applications initialize a table with test case function pointers using `unit_test*()` macros. This table is then passed to the `run_tests()` macro to execute the tests. `run_tests()` sets up the appropriate exception / signal handlers and other data structures prior to running each test function. When a unit test is complete `run_tests()` performs various checks to determine whether the test succeeded.

The second parameter to `run_tests()` is the prefix to the xUnit XML report in the following format: *prefix*\_xunit.xml.  For example, calling `run_tests(tests, "abcd")` will create an xUnit XML report called `abcd_xunit.xml`.

### Using run_tests()

#### run\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmockery.h>

// A test case that does nothing and succeeds.
void null_test_success(void **state) {
}

int main(int argc, char* argv[]) {
    const UnitTest tests[] = {
        unit_test(null_test_success),
    };
    return run_tests(tests, "run");
}
```

## Exception Handling
Before a test function is executed by run_tests(), exception / signal handlers are overridden with a handler that simply displays an error and exits a test function if an exception occurs. If an exception occurs outside of a test function, for example in Cmockery itself, the application aborts execution and returns an error code.

## Failure Conditions
If a failure occurs during a test function that's executed via run_tests(), the test function is aborted and the application's execution resumes with the next test function. Test failures are ultimately signalled via the Cmockery function fail(). The following events will result in the Cmockery library signalling a test failure...

## Assertions
Runtime assert macros like the standard C library's assert() should be redefined in modules being tested to use Cmockery's mock_assert() function. Normally mock_assert() signals a test failure. If a function is called using the expect_assert_failure() macro, any calls to mock_assert() within the function will result in the execution of the test. If no calls to mock_assert() occur during the function called via expect_assert_failure() a test failure is signalled.  This is easily handled by including `cmockery_override.h`.

### Using mock_assert()

#### assert\_module.c

```c
#include <assert.h>

// Override functions in the program to be tested
#if UNIT_TESTING
// Must be last include
#include <cmockery/cmockery_override.h>
#endif // UNIT_TESTING

void increment_value(int * const value) {
    assert(value);
    (*value) ++;
}

void decrement_value(int * const value) {
    if (value) {
        *value --;
    }
}
```

### assert\_module\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmockery.h>

extern void increment_value(int * const value);

/* This test case will fail but the assert is caught by run_tests() and the
 * next test is executed. */
void increment_value_fail(void **state) {
    increment_value(NULL);
}

// This test case succeeds since increment_value() asserts on the NULL pointer.
void increment_value_assert(void **state) {
    expect_assert_failure(increment_value(NULL));
}

/* This test case fails since decrement_value() doesn't assert on a NULL
 * pointer. */
void decrement_value_fail(void **state) {
    expect_assert_failure(decrement_value(NULL));
}

int main(int argc, char *argv[]) {
    const UnitTest tests[] = {
        unit_test(increment_value_fail),
        unit_test(increment_value_assert),
        unit_test(decrement_value_fail),
    };
    return run_tests(tests, "assert_module");
}
```

## Assert Macros

Cmockery provides an assortment of assert macros that tests applications should use use in preference to the C standard library's assert macro. On an assertion failure a Cmockery assert macro will write the failure to the standard error stream and signal a test failure. Due to limitations of the C language the general C standard library `assert()` and Cmockery's `assert_true()` and `assert_false()` macros can only display the expression that caused the assert failure. Cmockery's type specific assert macros, `assert_{type}_equal()` and `assert_{type}_not_equal()`, display the data that caused the assertion failure which increases data visibility aiding debugging of failing test cases.

### Using assert\_{type}\_equal() macros

#### assert\_macro.c

```c
#include <string.h>

static const char* status_code_strings[] = {
    "Address not found",
    "Connection dropped",
    "Connection timed out",
};

const char* get_status_code_string(const unsigned int status_code) {
    return status_code_strings[status_code];
};

unsigned int string_to_status_code(const char* const status_code_string) {
    unsigned int i;
    for (i = 0; i < sizeof(status_code_strings) /
                    sizeof(status_code_strings[0]); i++) {
        if (strcmp(status_code_strings[i], status_code_string) == 0) {
            return i;
        }
    }
    return ~0U;
}
```

#### assert\_macro\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmockery.h>

extern const char* get_status_code_string(const unsigned int status_code);
extern unsigned int string_to_status_code(
    const char* const status_code_string);

/* This test will fail since the string returned by get_status_code_string(0)
 * doesn't match "Connection timed out". */
void get_status_code_string_test(void **state) {
    assert_string_equal(get_status_code_string(0), "Address not found");
    assert_string_equal(get_status_code_string(1), "Connection timed out");
}

// This test will fail since the status code of "Connection timed out" isn't 1
void string_to_status_code_test(void **state) {
    assert_int_equal(string_to_status_code("Address not found"), 0);
    assert_int_equal(string_to_status_code("Connection timed out"), 1);
}

int main(int argc, char *argv[]) {
    const UnitTest tests[] = {
        unit_test(get_status_code_string_test),
        unit_test(string_to_status_code_test),
    };
    return run_tests(tests, "assert_macro");
}
```

## Dynamic Memory Allocation

To test for memory leaks, buffer overflows and underflows a module being tested by Cmockery should replace calls to `malloc()`, `calloc()` and `free()` to `test_malloc()`, `test_calloc()` and `test_free()` respectively. This can be handled by including `cmockery_override.h`. Each time a block is deallocated using `test_free()` it is checked for corruption, if a corrupt block is found a test failure is signalled. All blocks allocated using the `test_*()` allocation functions are tracked by the Cmockery library. When a test completes if any allocated blocks (memory leaks) remain they are reported and a test failure is signalled.

For simplicity Cmockery currently executes all tests in one process. Therefore all test cases in a test application share a single address space which means memory corruption from a single test case could potentially cause the test application to exit prematurely.

### Using Cmockery's Allocators

#### allocate\_module.c

```c
#include <malloc.h>
#ifdef UNIT_TESTING
/* Must be last include */
#include <cmockery/cmockery_override.h>
#endif

void leak_memory(void);
void buffer_overflow(void);
void buffer_underflow(void);

void leak_memory(void) {
    int * const temporary = (int*)malloc(sizeof(int));
    *temporary = 0;
}

void buffer_overflow(void) {
    char * const memory = (char*)malloc(sizeof(int));
    memory[sizeof(int)] = '!';
    free(memory);
}

void buffer_underflow(void) {
    char * const memory = (char*)malloc(sizeof(int));
    memory[-1] = '!';
    free(memory);
}
```

#### allocate\_module\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <inttypes.h>
#include <cmockery.h>

extern void leak_memory();
extern void buffer_overflow();
extern void buffer_underflow();

// Test case that fails as leak_memory() leaks a dynamically allocated block.
static void leak_memory_test(void **state) {
    leak_memory();
}

// Test case that fails as buffer_overflow() corrupts an allocated block.
static void buffer_overflow_test(void **state) {
    buffer_overflow();
}

// Test case that fails as buffer_underflow() corrupts an allocated block.
static void buffer_underflow_test(void **state) {
    buffer_underflow();
}

int main(int argc, char* argv[]) {
    const UnitTest tests[] = {
        unit_test(leak_memory_test),
        unit_test(buffer_overflow_test),
        unit_test(buffer_underflow_test),
    };
    return run_tests(tests, "allocate_module_test");
}
```

## Mock Functions

A unit test should ideally isolate the function or module being tested from any external dependencies. This can be performed using mock functions that are either statically or dynamically linked with the module being tested. Mock functions must be statically linked when the code being tested directly references external functions. Dynamic linking is simply the process of setting a function pointer in a table used by the tested module to reference a mock function defined in the unit test.

### Return Values

In order to simplify the implementation of mock functions Cmockery provides functionality which stores return values for mock functions in each test case using will_`return()`. These values are then returned by each mock function using calls to `mock()`. Values passed to `will_return()` are added to a queue for each function specified. Each successive call to `mock()` from a function removes a return value from the queue. This makes it possible for a mock function to use multiple calls to `mock()` to return output parameters in addition to a return value. In addition this allows the specification of return values for multiple calls to a mock function.

#### Using will\_return()

##### database.h

```c
typedef struct DatabaseConnection DatabaseConnection;

/* Function that takes an SQL query string and sets results to an array of
 * pointers with the result of the query.  The value returned specifies the
 * number of items in the returned array of results.  The returned array of
 * results are statically allocated and should not be deallocated using free()
 */
typedef unsigned int (*QueryDatabase)(
    DatabaseConnection* const connection, const char * const query_string,
    void *** const results);

// Connection to a database.
struct DatabaseConnection {
    const char *url;
    unsigned int port;
    QueryDatabase query_database;
};
```

##### customer\_database.c

```c
#include <stdint.h>
#include <stddef.h>
#include <stdio.h>
#include <database.h>
#ifdef _WIN32
#define snprintf _snprintf
#endif // _WIN32

DatabaseConnection* connect_to_customer_database(void);
unsigned int get_customer_id_by_name(
        DatabaseConnection * const connection,
        const char * const customer_name);

// Connect to the database containing customer information.
DatabaseConnection* connect_to_customer_database(void) {
    return connect_to_database("customers.abcd.org", 321);
}

/* Find the ID of a customer by his/her name returning a value > 0 if
 * successful, 0 otherwise. */
unsigned int get_customer_id_by_name(
        DatabaseConnection * const connection,
        const char * const customer_name) {
    char query_string[256];
    int number_of_results;
    void **results;
    snprintf(query_string, sizeof(query_string),
             "SELECT ID FROM CUSTOMERS WHERE NAME = %s", customer_name);
    number_of_results = connection->query_database(connection, query_string,
                                                   &results);
    if (number_of_results != 1) {
        return -1;
    }
    return (uintptr_t)results[0];
}
```

##### customer\_database\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <inttypes.h>
#include <cmockery.h>
#include <database.h>

extern DatabaseConnection* connect_to_customer_database();
extern unsigned int get_customer_id_by_name(
    DatabaseConnection * const connection, const char * const customer_name);

// Mock query database function.
static unsigned int mock_query_database(
        DatabaseConnection* const connection, const char * const query_string,
        void *** const results) {
    *results = (void**)((uintptr_t)mock());
    return (unsigned int)mock();
}

// Mock of the connect to database function.
DatabaseConnection* connect_to_database(const char * const database_url,
                                        const unsigned int port) {
    return (DatabaseConnection*)((uintptr_t)mock());
}

static void test_connect_to_customer_database(void **state) {
    will_return(connect_to_database, 0x0DA7ABA53);
    assert_int_equal((uintptr_t)connect_to_customer_database(), 0x0DA7ABA53);
}

/* This test fails as the mock function connect_to_database() will have no
 * value to return. */
static void test_fail_connect_to_customer_database(void **state) {
    assert_true(connect_to_customer_database() ==
                (DatabaseConnection*)0x0DA7ABA53);
}

static void test_get_customer_id_by_name(void **state) {
    DatabaseConnection connection = {
        "somedatabase.somewhere.com", 12345678, mock_query_database
    };
    // Return a single customer ID when mock_query_database() is called.
    int customer_ids = 543;
    will_return(mock_query_database, &customer_ids);
    will_return(mock_query_database, 1);
    assert_int_equal(get_customer_id_by_name(&connection, "john doe"), 543);
}

int main(void) {
    const UnitTest tests[] = {
        unit_test(test_connect_to_customer_database),
        unit_test(test_get_customer_id_by_name),
	unit_test(test_fail_connect_to_customer_database),
    };
    return run_tests(tests, "customer_database");
}
```



## Checking Parameters

In addition to storing the return values of mock functions, Cmockery provides functionality to store expected values for mock function parameters using the `expect_*()` functions provided. A mock function parameter can then be validated using the `check_expected()` macro.

Successive calls to `expect_*()` macros for a parameter queues values to check the specified parameter. `check_expected()` checks a function parameter against the next value queued using `expect_*()`, if the parameter check fails a test failure is signalled. In addition if `check_expected()` is called and no more parameter values are queued a test failure occurs.

### Using expect\_\*()

#### product\_database.c

```c
#include <database.h>

DatabaseConnection* connect_to_product_database(void);

// Connect to the database containing customer information.
DatabaseConnection* connect_to_product_database(void) {
    return connect_to_database("products.abcd.org", 322);
}
```

#### product\_database\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <inttypes.h>
#include <cmockery.h>
#include <database.h>

extern DatabaseConnection* connect_to_product_database(void);

/* Mock connect to database function.
 * NOTE: This mock function is very general could be shared between tests
 * that use the imaginary database.h module. */
DatabaseConnection* connect_to_database(const char * const url,
                                        const unsigned int port) {
    check_expected(url);
    check_expected(port);
    return (DatabaseConnection*)((uintptr_t)mock());
}

static void test_connect_to_product_database(void **state) {
    expect_string(connect_to_database, url, "products.abcd.org");
    expect_value(connect_to_database, port, 322);
    will_return(connect_to_database, 0xDA7ABA53);
    assert_int_equal((uintptr_t)connect_to_product_database(), 0xDA7ABA53);
}

/* This test will fail since the expected URL is different to the URL that is
 * passed to connect_to_database() by connect_to_product_database(). */
static void test_connect_to_product_database_bad_url(void **state) {
    expect_string(connect_to_database, url, "products.abcd.com");
    expect_value(connect_to_database, port, 322);
    will_return(connect_to_database, 0xDA7ABA53);
    assert_int_equal((uintptr_t)connect_to_product_database(), 0xDA7ABA53);
}

/* This test will fail since the mock connect_to_database() will attempt to
 * retrieve a value for the parameter port which isn't specified by this
 * test function. */
static void test_connect_to_product_database_missing_parameter(void **state) {
    expect_string(connect_to_database, url, "products.abcd.org");
    will_return(connect_to_database, 0xDA7ABA53);
    assert_int_equal((uintptr_t)connect_to_product_database(), 0xDA7ABA53);
}

int main(void) {
    const UnitTest tests[] = {
        unit_test(test_connect_to_product_database),
        unit_test(test_connect_to_product_database_bad_url),
        unit_test(test_connect_to_product_database_missing_parameter),
    };
    return run_tests(tests, "product_database");
}
```

## Test State

Cmockery allows the specification of multiple setup and tear down functions for each test case. Setup functions, specified by the unit_test_setup() or unit_test_setup_teardown() macros allow common initialization to be shared between multiple test cases. In addition, tear down functions, specified by the unit_test_teardown() or unit_test_setup_teardown() macros provide a code path that is always executed for a test case even when it fails.

### Using unit\_test\_setup\_teardown()

#### key\_value.c

```c
#include <stddef.h>
#include <stdlib.h>
#include <string.h>

#include "key_value.h"

static KeyValue *key_values = NULL;
static unsigned int number_of_key_values = 0;

void set_key_values(KeyValue * const new_key_values,
                    const unsigned int new_number_of_key_values) {
    key_values = new_key_values;
    number_of_key_values = new_number_of_key_values;
}

// Compare two key members of KeyValue structures.
static int key_value_compare_keys(const void *a, const void *b) {
    return (int)((KeyValue*)a)->key - (int)((KeyValue*)b)->key;
}

// Search an array of key value pairs for the item with the specified value.
KeyValue* find_item_by_value(const char * const value) {
  unsigned int i;
    for (i = 0; i < number_of_key_values; i++) {
        if (strcmp(key_values[i].value, value) == 0) {
            return &key_values[i];
        }
    }
    return NULL;
}

// Sort an array of key value pairs by key.
void sort_items_by_key(void) {
    qsort(key_values, number_of_key_values, sizeof(*key_values),
          key_value_compare_keys);
}
```

#### key\_value\_test.c

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <string.h>
#include <inttypes.h>
#include <cmockery.h>

#include "key_value.h"

static KeyValue key_values[] = {
    { 10, "this" },
    { 52, "test" },
    { 20, "a" },
    { 13, "is" },
};

static void create_key_values(void **state) {
    KeyValue * const items = (KeyValue*)test_malloc(sizeof(key_values));
    memcpy(items, key_values, sizeof(key_values));
    *state = (void*)items;
    set_key_values(items, sizeof(key_values) / sizeof(key_values[0]));
}

static void destroy_key_values(void **state) {
    test_free(*state);
    set_key_values(NULL, 0);
}

static void test_find_item_by_value(void **state) {
    unsigned int i;
    for (i = 0; i < sizeof(key_values) / sizeof(key_values[0]); i++) {
        KeyValue * const found  = find_item_by_value(key_values[i].value);
        assert_true(found != NULL);
        assert_int_equal(found->key, key_values[i].key);
        assert_string_equal(found->value, key_values[i].value);
    }
}

static void test_sort_items_by_key(void **state) {
    unsigned int i;
    KeyValue * const kv = *state;
    sort_items_by_key();
    for (i = 1; i < sizeof(key_values) / sizeof(key_values[0]); i++) {
        assert_true(kv[i - 1].key < kv[i].key);
    }
}

int main(void) {
    const UnitTest tests[] = {
        unit_test_setup_teardown(test_find_item_by_value, create_key_values,
                                 destroy_key_values),
        unit_test_setup_teardown(test_sort_items_by_key, create_key_values,
                                 destroy_key_values),
    };
    return run_tests(tests, "key_value");
}
```

## Example

A small command line calculator `calculator.c` application and test application that full exercises the calculator application `calculator_test.c` are provided as an example of Cmockery's features discussed in this document.
