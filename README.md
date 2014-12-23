
Minimal Asynchronous Logger (MAL)
-----------
A non feature-bloated asynchronous data logger. Sponsored by my employer **Diadrom AB.**

We just wanted an asynchronous logger that can be used from many dynamically loaded libraries without doing link-time hacks like linking static and hiding symbols and some other "niceties".

After having maintained a slightly modified version of google log and given the fact that this is a very small project we decided that existing wheels weren't round enough.

## Design rationale ##

 - Simple. Not over abstracted and feature bloated, explicit, easy to figure out what the code is doing, easy to modify.
 - Low latency, fast for the caller.
 - Asynchronous (synchronous calls can be made on top for special messages, but they are way slower than using a synchronous logger in the first place).
 - Minimum string formatting in the calling thread for the most common use cases.
 - Keeps ordering between threads.
 - Doesn't use thread-local-storage, the user threads are assumed as external and no extra info is attached to them.

## Various features ##

 - Targeting g++4.7 and VS 2010
 - Boost dependencies just for parts that will eventually go to the C++ standard.
 - No singleton by design, usable from dynamically loaded libraries. The user provides the instance either explicitly or by a global function (Koenig lookup).
 - Suitable for soft-realtime work. Once it's initialized the fast-path can be clear from heap allocations if properly configured.
 - File rotation-slicing (needs some external help at initialization time until std::filesystem isn't implemented in some compilers, see below).
 - One conditional call overhead for inactive severities.
 - Lazy parameter evaluation (as usual with most logging libraries).
 - No ostreams (a very ugly part of C++ for my liking), just format strings checked at compile time (if the compiler supports it) with type safe values. An on-stack ostream adapter is available as alast resort, but its use is more verbose and has more overhead.
 - The log severity can be externally changed outside of the process. The IPC mechanism is the simplest, the log worker periodically polls some file descriptors when idle (if configured to).
 
## How does it work ##

It just borrows ideas from many of the loggers out there.

When the user is to write a log message, the size is precomputed, then the memory is reclaimed either from the fixed size pool or the heap (configurable) and then message is passed to the log queue in an internal format.

Most of the time the user thread doesn't format strings, just encodes built-in type values (integers are variable-length encoded) and whole program duration C strings. Deep copies can be done if required too.

The messages are formatted by using printf-style strings, where the formatting string is required to be a literal (not a const char*, this is a limitation of C++11 constexpr), e.g:

log_error ("the value of i is {} and the value of j is {}", i, j);

The function is type-safe, when "constexpr" and variadic template parameters are available the format string is matched with the parameters at compile time, otherwise the errors are caught at run time in the logging output.

> see this [example](https://github.com/RafaGago/mini-async-log/blob/master/example/overview.cpp) that more or less shows all available features.

## Benchmarks ##

I used to have some benchmarks here showing that "my new thing (TM)" is the best invention since sliced-bread, but as the benchmarks were mostly an apples-to-oranges comparison (e.g. comparing with glog which is half-synchronous) I just removed them.

With the actual performance and if the logger is configured to timestamp at the client/producer side it takes the same or more time to retrieve the timestamp with the std::chrono/boost::chrono library than to build and enqueue a simple message composed of e.g. a format string and three integers.

> [These were the benchmarks that I had](https://github.com/RafaGago/mini-async-log/blob/master/example/benchmark.cpp).

## File rotation ##

The library can rotate fixed size log files.

Using the current C++11 standard files can just be created, modified and deleted. There is no way to list a directory, so the user is required to pass at start time the list of files generated by previous runs. I may add support for boost::filesystem/std::filesystem, but just as an optional but ready external code, so everyone can skip this heavy dependency. There is an example using boost::filesystem in the "/extras" folder

> There is an [example](https://github.com/RafaGago/mini-async-log/blob/master/example/rotation.cpp) here.

## Initialization ##

The library isn't a singleton, so the user should provide the logger instance. Even of many modules call the initialization function only one of them will succeed.

There are two methods to enqueue a log entry, one is to provide it explicitly and the other one is by accessing a global function.

If no instance is provided, the global function "get_mal_logger_instance()" will be called without being namespace qualified, so you can use Koenig lookup/ADL. This happens when the user calls the macros with no suffix, as e.g. "log_error(fmt string, ...)".

To provide the instance explictly the macros with the "_i" function need to be called, e.g. "log_error_i(instance, fmt_string, ...)"

The name of the function can be changed at compile time, by defining MAL_GET_LOGGER_INSTANCE_FUNCNAME.

Be aware that it's dangerous to have a dynamic library or executable loaded multiple times logging to the same folder and rotating files each other. Workarounds exists, you can prepend the folder name with the process name and ID, disable rotation and manage rotation externally (e.g. by using logrotate), etc.

## Termination ##

The worker blocks in its destructor until its work queue is empty when normally exiting a program.

When a signal is sent you can call the frontend function  [on termination](https://github.com/RafaGago/mini-async-log/blob/master/include/mal_log/frontend.hpp). This will early interrupt any synchronous calls you made.

## Errors ##

As for now, every function returns a boolean if it succeeded or false if it didn't. A filtered out/below severity call returns true.

The only possible failures are either to be unable to allocate memory for a log entry, an asynchronous call that was interrupted by "on_termination" or a string byte that was to be deep copied but would overflow the length variable (mostly a bug, I highly doubt that someone will log messages that take 4GB).

The functions never throw.

## Weaknesses ##

 1. Just ASCII.
 2. No C++ ostream support. (not sure if it's a good or a bad thing...). Swapping logger in an existing codebase may not be worth the effort in some cases. Printing some classes that have overloaded the stream operator can be repetitive (I have to find a solution for this).
 3. Limited formatting abilities (it can be improved with more parser complexity).
 4. No way to output runtime strings/memory regions without deep-copying them.
 5. Ugly macros, but unfortunately the same syntax can't be achieved in any other way.
 6. Format string need to be literals. A const char* isn't enough (constexpr can't iterate them at compile time).
 
The third point is the most restrictive for my liking, it's just inherent to the asynchronous/non-blocking design, there is no guarantee about the passed data lifetime.

It's possible to artificially increment the refcount of a shared_ptr by copying it to an instance created using "placement_new" and to decrement it in the worker using the same trick, I keep this idea on hold for now.


## Compiler macros ##

Those that are self-explanatory won't be explained.

 - *MAL_GET_LOGGER_INSTANCE_FUNC*: See the "Initialization" chapter above.
 - *MAL_STRIP_LOG_SEVERITY*: Removes the entries of this severity and below at compile time. 0 is the "debug" severity, 5 is the "critical" severity. Stripping at level 5 leaves no log entries at all. Yo can define e.g. MAL_STRIP_LOG_DEBUG, MAL_STRIP_LOG_TRACE, etc. instead. If you define MAL_STRIP_LOG_TRACE all the severities below will be automatically defined for you (in this case MAL_STRIP_LOG_DEBUG).
 - *MAL_DYNLIB_COMPILE*: Define it when compiling as a library (Windows).
 - *MAL_CACHE_LINE_SIZE*: The cache line size of the machine you are compiling for. This is just used for data structure padding. 64 is defaulted when undefined.
 - *MAL_USE_BOOST_CSTDINT*: If your compiler doesn't have <cstdint> use boost.
 - *MAL_USE_BOOST_ATOMIC*
 - *MAL_USE_BOOST_CHRONO*
 - *MAL_USE_BOOST_THREAD*
 - *MAL_NO_VARIABLE_INTEGER_WIDTH*: Integers are encoded ignoring the number trailing bytes set to zero, not based on its data type size. So when this isn't defined e.g. encoding an uint64 with a value up to 255 takes one byte (plus 1 byte header). Otherwise all uint64 values will take 8 bytes (plus header), so encoding is less space efficient in this way but it frees the CPU and allows the compiler to inline more.
 
## Using the library ##

You can compile the files in the "src" folder and make a library or just use compile everything in your project.

If you want to compile the library inside your project you need to merge the "src" and "include" folders (or to add both as an include directory to the compiler) and to compile "frontend_def.hpp" in one translation unit.

Otherwise you can use the makefile in the /build/linux folder, one example of command invocation could be:

    make CXXFLAGS="-DNDEBUG -DMAL_USE_BOOST_THREAD -DMAL_USE_BOOST_CHRONO -DBOOST_ALL_DYN_LINK -DBOOST_CHRONO_HEADER_ONLY" LDLIBS="-lboost_thread" CXX="arm-linux-gnueabihf-g++"

## Todo ##

 - Explain the .sln to be able to build on Windows.

> Written with [StackEdit](https://stackedit.io/).

