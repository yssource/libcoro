# libcoro C++20 linux coroutine library

[![CI](https://github.com/jbaldwin/libcoro/workflows/build/badge.svg)](https://github.com/jbaldwin/libcoro/workflows/build/badge.svg)
[![Coverage Status](https://coveralls.io/repos/github/jbaldwin/libcoro/badge.svg?branch=master)](https://coveralls.io/github/jbaldwin/libcoro?branch=master)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/c190d4920e6749d4b4d1a9d7d6687f4f)](https://www.codacy.com/gh/jbaldwin/libcoro/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=jbaldwin/libcoro&amp;utm_campaign=Badge_Grade)
[![language][badge.language]][language]
[![license][badge.license]][license]

**libcoro** is licensed under the Apache 2.0 license.

**libcoro** is meant to provide low level coroutine constructs for building larger applications, the current focus is around high performance networking coroutine support.

## Overview
* C++20 coroutines!
* Modern Safe C++20 API
* Higher level coroutine constructs
    - coro::task<T>
    - coro::generator<T>
    - coro::event
    - coro::latch
    - coro::mutex
    - coro::sync_wait(awaitable)
        - coro::when_all(awaitable...) -> coro::task<T>...
        - coro::when_all_results(awaitable...) -> T... (Future)
* Schedulers
    - coro::thread_pool for coroutine cooperative multitasking
    - coro::io_scheduler for driving i/o events, uses thread_pool for coroutine execution
        - epoll driver
        - io_uring driver (Future, will be required for async file i/o)
* Coroutine Networking
    - coro::net::dns_resolver for async dns, leverages libc-ares
    - coro::net::tcp_client and coro::net::tcp_server
    - coro::net::udp_peer

### A note on co_await
Its important to note with coroutines that depending on the construct used _any_ `co_await` has the potential to switch the thread that is executing the currently running coroutine.  In general this shouldn't affect the way any user of the library would write code except for `thread_local`.  Usage of `thread_local` should be extremely careful and _never_ used across any `co_await` boundary do to thread switching and work stealing on thread pools.


### coro::generator<T>
The `coro::generator<T>` construct is a coroutine which can generate one or more values.

```C++
#include <coro/coro.hpp>
#include <iostream>

int main()
{
    auto task = [](uint64_t count_to) -> coro::task<void> {
        // Create a generator function that will yield and incrementing
        // number each time its called.
        auto gen = []() -> coro::generator<uint64_t> {
            uint64_t i = 0;
            while (true)
            {
                co_yield i++;
            }
        };

        // Generate the next number until its greater than count to.
        for (auto val : gen())
        {
            std::cout << val << ", ";

            if (val >= count_to)
            {
                break;
            }
        }
        co_return;
    };

    coro::sync_wait(task(100));
}
```

Expected output:
```bash
$ ./examples/coro_generator
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100,
```

### coro::event
The `coro::event` is a thread safe async tool to have 1 or more waiters suspend for an event to be set before proceeding.  The implementation of event currently will resume execution of all waiters on the thread that sets the event.  If the event is already set when a waiter goes to wait on the thread they will simply continue executing with no suspend or wait time incurred.

```C++
#include <coro/coro.hpp>
#include <iostream>

int main()
{
    coro::event e;

    // These tasks will wait until the given event has been set before advancing.
    auto make_wait_task = [](const coro::event& e, uint64_t i) -> coro::task<void> {
        std::cout << "task " << i << " is waiting on the event...\n";
        co_await e;
        std::cout << "task " << i << " event triggered, now resuming.\n";
        co_return;
    };

    // This task will trigger the event allowing all waiting tasks to proceed.
    auto make_set_task = [](coro::event& e) -> coro::task<void> {
        std::cout << "set task is triggering the event\n";
        e.set();
        co_return;
    };

    // Given more than a single task to synchronously wait on, use when_all() to execute all the
    // tasks concurrently on this thread and then sync_wait() for them all to complete.
    coro::sync_wait(coro::when_all(make_wait_task(e, 1), make_wait_task(e, 2), make_wait_task(e, 3), make_set_task(e)));
}
```

Expected output:
```bash
$ ./examples/coro_event
task 1 is waiting on the event...
task 2 is waiting on the event...
task 3 is waiting on the event...
set task is triggering the event
task 3 event triggered, now resuming.
task 2 event triggered, now resuming.
task 1 event triggered, now resuming.
```

### coro::latch
The `coro::latch` is a thread safe async tool to have 1 waiter suspend until all outstanding events have completed before proceeding.

```C++
#include <coro/coro.hpp>
#include <iostream>

int main()
{
    // Complete worker tasks faster on a thread pool, using the io_scheduler version so the worker
    // tasks can yield for a specific amount of time to mimic difficult work.  The pool is only
    // setup with a single thread to showcase yield_for().
    coro::io_scheduler tp{coro::io_scheduler::options{.pool = coro::thread_pool::options{.thread_count = 1}}};

    // This task will wait until the given latch setters have completed.
    auto make_latch_task = [](coro::latch& l) -> coro::task<void> {
        // It seems like the dependent worker tasks could be created here, but in that case it would
        // be superior to simply do: `co_await coro::when_all(tasks);`
        // It is also important to note that the last dependent task will resume the waiting latch
        // task prior to actually completing -- thus the dependent task's frame could be destroyed
        // by the latch task completing before it gets a chance to finish after calling resume() on
        // the latch task!

        std::cout << "latch task is now waiting on all children tasks...\n";
        co_await l;
        std::cout << "latch task dependency tasks completed, resuming.\n";
        co_return;
    };

    // This task does 'work' and counts down on the latch when completed.  The final child task to
    // complete will end up resuming the latch task when the latch's count reaches zero.
    auto make_worker_task = [](coro::io_scheduler& tp, coro::latch& l, int64_t i) -> coro::task<void> {
        // Schedule the worker task onto the thread pool.
        co_await tp.schedule();
        std::cout << "worker task " << i << " is working...\n";
        // Do some expensive calculations, yield to mimic work...!  Its also important to never use
        // std::this_thread::sleep_for() within the context of coroutines, it will block the thread
        // and other tasks that are ready to execute will be blocked.
        co_await tp.yield_for(std::chrono::milliseconds{i * 20});
        std::cout << "worker task " << i << " is done, counting down on the latch\n";
        l.count_down();
        co_return;
    };

    const int64_t                 num_tasks{5};
    coro::latch                   l{num_tasks};
    std::vector<coro::task<void>> tasks{};

    // Make the latch task first so it correctly waits for all worker tasks to count down.
    tasks.emplace_back(make_latch_task(l));
    for (int64_t i = 1; i <= num_tasks; ++i)
    {
        tasks.emplace_back(make_worker_task(tp, l, i));
    }

    // Wait for all tasks to complete.
    coro::sync_wait(coro::when_all(tasks));
}
```

Expected output:
```bash
$ ./examples/coro_latch
latch task is now waiting on all children tasks...
worker task 1 is working...
worker task 2 is working...
worker task 3 is working...
worker task 4 is working...
worker task 5 is working...
worker task 1 is done, counting down on the latch
worker task 2 is done, counting down on the latch
worker task 3 is done, counting down on the latch
worker task 4 is done, counting down on the latch
worker task 5 is done, counting down on the latch
latch task dependency tasks completed, resuming.
```

### coro::mutex
The `coro::mutex` is a thread safe async tool to protect critical sections and only allow a single thread to execute the critical section at any given time.  Mutexes that are uncontended are a simple CAS operation with a memory fence 'acquire' to behave similar to `std::mutex`.  If the lock is contended then the thread will add itself to a FIFO queue of waiters and yield excution to allow another coroutine to process on that thread while it waits to acquire the lock.

Its important to note that upon releasing the mutex that thread will immediately start processing the next waiter in line for the `coro::mutex`, the mutex is only unlocked/released once all waiters have been processed.  This guarantees fair execution in a FIFO manner, but it also means all coroutines that stack in the waiter queue will end up shifting to the single thread that is executing all waiting coroutines.  It is possible to reschedule after the critical section onto a thread pool to re-distribute the work.  Perhaps an auto-reschedule on a given thread pool is a good feature to implement in the future to prevent this behavior so the post critical section work in the coroutines is redistributed amongst all available thread pool threads.

```C++
#include <coro/coro.hpp>
#include <iostream>

int main()
{
    coro::thread_pool     tp{coro::thread_pool::options{.thread_count = 4}};
    std::vector<uint64_t> output{};
    coro::mutex           mutex;

    auto make_critical_section_task = [&](uint64_t i) -> coro::task<void> {
        co_await tp.schedule();
        // To acquire a mutex lock co_await its lock() function.  Upon acquiring the lock the
        // lock() function returns a coro::scoped_lock that holds the mutex and automatically
        // unlocks the mutex upon destruction.  This behaves just like std::scoped_lock.
        {
            auto scoped_lock = co_await mutex.lock();
            output.emplace_back(i);
        } // <-- scoped lock unlocks the mutex here.
        co_return;
    };

    const size_t                  num_tasks{100};
    std::vector<coro::task<void>> tasks{};
    tasks.reserve(num_tasks);
    for (size_t i = 1; i <= num_tasks; ++i)
    {
        tasks.emplace_back(make_critical_section_task(i));
    }

    coro::sync_wait(coro::when_all(tasks));

    // The output will be variable per run depending on how the tasks are picked up on the
    // thread pool workers.
    for (const auto& value : output)
    {
        std::cout << value << ", ";
    }
}
```

Expected output, note that the output will vary from run to run based on how the thread pool workers
are scheduled and in what order they acquire the mutex lock:
```bash
$ ./examples/coro_mutex
1, 2, 3, 4, 5, 6, 7, 8, 10, 9, 12, 11, 13, 14, 15, 16, 17, 18, 19, 21, 22, 20, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 47, 48, 49, 46, 50, 51, 52, 53, 54, 55, 57, 58, 59, 56, 60, 62, 61, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100,
```

## Usage

### Requirements
    C++20 Compiler with coroutine support
        g++10.2 is tested
    CMake
    make or ninja
    pthreads
    gcov/lcov (For generating coverage only)

### Instructions

#### Cloning the project
This project uses gitsubmodules, to properly checkout this project use:

    git clone --recurse-submodules <libcoro-url>

This project depends on the following projects:
 * [libc-ares](https://github.com/c-ares/c-ares) For async DNS resolver.
 * [catch2](https://github.com/catchorg/Catch2) For testing.

#### Building
    mkdir Release && cd Release
    cmake -DCMAKE_BUILD_TYPE=Release ..
    cmake --build .

CMake Options:

| Name                   | Default | Description                                                   |
|:-----------------------|:--------|:--------------------------------------------------------------|
| LIBCORO_BUILD_TESTS    | ON      | Should the tests be built?                                    |
| LIBCORO_CODE_COVERAGE  | OFF     | Should code coverage be enabled? Requires tests to be enabled |
| LIBCORO_BUILD_EXAMPLES | ON      | Should the examples be built?                                 |

#### Adding to your project

##### add_subdirectory()

```cmake
# Include the checked out libcoro code in your CMakeLists.txt file
add_subdirectory(path/to/libcoro)

# Link the libcoro cmake target to your project(s).
target_link_libraries(${PROJECT_NAME} PUBLIC libcoro)

```

##### FetchContent
CMake can include the project directly by downloading the source, compiling and linking to your project via FetchContent, below is an example on how you might do this within your project.


```cmake
cmake_minimum_required(VERSION 3.11)

# Fetch the project and make it available for use.
include(FetchContent)
FetchContent_Declare(
    libcoro
    GIT_REPOSITORY https://github.com/jbaldwin/libcoro.git
    GIT_TAG        <TAG_OR_GIT_HASH>
)
FetchContent_MakeAvailable(libcoro)

# Link the libcoro cmake target to your project(s).
target_link_libraries(${PROJECT_NAME} PUBLIC libcoro)

```

#### Tests
The tests will automatically be run by github actions on creating a pull request.  They can also be ran locally:

    # Invoke via cmake with all output from the tests displayed to console:
    ctest -VV

    # Or invoke directly, can pass the name of tests to execute, the framework used is catch2.
    # Tests are tagged with their group, below is howt o run all of the coro::net::tcp_server tests:
    ./Debug/test/libcoro_test "[tcp_server]"

### Support

File bug reports, feature requests and questions using [GitHub libcoro Issues](https://github.com/jbaldwin/libcoro/issues)

Copyright © 2020-2021 Josh Baldwin

[badge.language]: https://img.shields.io/badge/language-C%2B%2B20-yellow.svg
[badge.license]: https://img.shields.io/badge/license-Apache--2.0-blue

[language]: https://en.wikipedia.org/wiki/C%2B%2B17
[license]: https://en.wikipedia.org/wiki/Apache_License
