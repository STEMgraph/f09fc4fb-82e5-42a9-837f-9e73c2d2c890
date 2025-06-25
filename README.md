<!---
{
  "id": "f09fc4fb-82e5-42a9-837f-9e73c2d2c890",
  "depends_on": [],
  "author": "Stephan Bökelmann",
  "first_used": "2025-06-23",
  "keywords": ["redis", "cpp", "rediscpp", "CMake FetchContent", "streams", "pubsub"]
}
--->

# Working with Redis in C++ using **redis-cpp**

> In this exercise you will learn how to use the **redis-cpp** header-only library to interact with Redis for counting keys, consuming streams, and using publish/subscribe patterns. Furthermore we will explore integrating the library via CMake’s FetchContent, handling asynchronous data, and designing small, robust clients.

## Introduction

This exercise sheet guides you through three common Redis patterns using the **redis-cpp** C++17 header-only client. You already know the basics of Redis and how to use CMake’s FetchContent to pull in dependencies; here you’ll see how to wire everything together, write a loop that atomically increments a counter, consume a Redis Stream as events arrive, and build a simple publish/subscribe client.

At its core, **redis-cpp** offers:

* **`make_stream(host, port)`**: Establishes a TCP connection to the Redis server and returns a `std::shared_ptr<rediscpp::Stream>`.
* **`execute(stream, args...)`**: Sends a Redis command and returns a `rediscpp::Response` that you convert to your desired type via `.as<T>()` or inspect with helper methods (e.g., `is_null()`, `as_array()`).
* **Blocking and non-blocking operations**: Commands like `XREAD BLOCK` or `SUBSCRIBE` utilize blocking reads on the underlying socket in a straightforward manner.

Each task below pairs a CMake configuration with C++ code. Before each code listing, you’ll find detailed comments explaining each line.

### Further Readings and Other Sources

1. **redis-cpp GitHub repository** (examples & API):
   [https://github.com/tdv/redis-cpp](https://github.com/tdv/redis-cpp)
2. **Redis Streams documentation**:
   [https://redis.io/topics/streams](https://redis.io/topics/streams)
3. **Redis Publish/Subscribe**:
   [https://redis.io/topics/pubsub](https://redis.io/topics/pubsub)
4. **CMake FetchContent module**:
   [https://cmake.org/cmake/help/latest/module/FetchContent.html](https://cmake.org/cmake/help/latest/module/FetchContent.html)
5. **Official Redis Introduction**:
   [https://redis.io/topics/introduction](https://redis.io/topics/introduction)

---

## Tasks
### Task 0: Ensure that Redis is installed

Make sure you have installed a Redis version newer than 5.0.
Verify this by running:

```sh
redis-server --version
```

If Redis is not yet installed, set up the default installation of Redis using your package manager:

```sh
sudo apt update
sudo apt install redis -y
```


### Task 1: Auto-increment a Key Every Second

**Goal:** Use C++ to increment a Redis key (`counter`) every second, then verify the value externally with `redis-cli`.

1. **CMake Configuration**

   ```cmake
   cmake_minimum_required(VERSION 3.14)
   project(MyRedisProject LANGUAGES CXX)

   # 1. Load FetchContent to pull in external projects
   include(FetchContent)

   # 2. Declare the redis-cpp dependency (header-only)
   FetchContent_Declare(
     rediscpp
     GIT_REPOSITORY https://github.com/tdv/redis-cpp.git
     GIT_TAG        master
   )

   # 3. Fetch & make available the library target 'rediscpp'
   FetchContent_MakeAvailable(rediscpp)

   # 4. Link to threading library for sleep operations
   find_package(Threads REQUIRED)

   target_include_directories(increment
     PRIVATE
       ${rediscpp_SOURCE_DIR}/include  # Path to redis-cpp headers
   )

   # 5. Define executable for Task 1
   add_executable(increment increment.cpp)
   target_link_libraries(increment
     PRIVATE
       Threads::Threads  # For std::this_thread::sleep
   )
   ```

   * **Steps 1–3** handle pulling in **redis-cpp**, making it a valid target.
   * **Step 4** ensures you can use `<thread>` for timing loops.
   * **Step 5** creates the `increment` binary and links necessary targets.

2. **`increment.cpp` Explanation**

   ```cpp
   // Include timing and I/O utilities
   #include <chrono>    // std::chrono::seconds
   #include <iostream>  // std::cout, std::cerr
   #include <thread>    // std::this_thread::sleep_for

   // Enable header-only mode (no separate library)
   #define REDISCPP_HEADER_ONLY
   // Core redis-cpp headers
   #include <redis-cpp/stream.h>   // rediscpp::make_stream
   #include <redis-cpp/execute.h>  // rediscpp::execute

   int main() {
       try {
           // 1. Connect to Redis at localhost:6379
           auto stream = rediscpp::make_stream("localhost", "6379");

           // 2. Loop indefinitely
           while (true) {
               // 3. Send INCR command for key "counter"
               auto resp = rediscpp::execute(*stream, "INCR", "counter");
               // 4. Convert response to integer and print
               std::cout << "New counter value: "
                         << resp.as<long long>() << std::endl;

               // 5. Wait for 1 second before next increment
               std::this_thread::sleep_for(std::chrono::seconds(1));
           }
       } catch (const std::exception &e) {
           // Error handling: e.what() holds the error message
           std::cerr << "Error: " << e.what() << std::endl;
           return EXIT_FAILURE;
       }
       return EXIT_SUCCESS;
   }
   ```

   * **Line 1–3**: Include standard headers for timing and I/O.
   * **Macro `REDISCPP_HEADER_ONLY`**: Instructs **redis-cpp** to compile purely from headers without external linking.
   * **`make_stream`**: Opens a TCP connection; throws on failure.
   * **`execute(*stream, ...)`**: Issues a Redis command. Here `INCR counter` atomically increments by 1.
   * **`resp.as<long long>()`**: Extracts the new integer value.
   * **Error block**: Catches socket errors or protocol issues, printing details.

3. **Compile & Run**

   ```bash
   cmake -B ./build       # Configure with our CMakeLists.txt
   camke --build ./build  # Build the 'increment' binary
   ./build/increment      # Starts printing counter values each second
   ```

4. **External Verification**
   In a separate shell:

   ```bash
   redis-cli GET counter  # Should match the latest printed value
   ```

---

### Task 2: Consuming a Redis Stream in C++

**Goal:** Read events from a Redis Stream (`event_stream`) in real time and decode field/value pairs.

1. **Stream Setup**

   ```bash
   redis-cli XADD event_stream '*' message "startup"
   ```

   * **`XADD event_stream * ...`**: Creates `event_stream` if absent and appends one entry with auto-generated ID.

2. **`stream_consumer.cpp` Explanation**

   ```cpp
   #include <iostream>  // std::cout, std::cerr
   #include <string>    // std::string

   #define REDISCPP_HEADER_ONLY
   #include <redis-cpp/stream.h>
   #include <redis-cpp/execute.h>

   int main() {
       try {
           // 1. Connect to Redis
           auto stream = rediscpp::make_stream("localhost", "6379");
           std::string last_id = "0";  // Start at the beginning

           while (true) {
               // 2. BLOCK for up to 5 seconds for new records after last_id
               auto resp = rediscpp::execute(*stream,
                   "XREAD", "BLOCK", "5000", "STREAMS",
                   "event_stream", last_id);

               // 3. If no new data, resp.is_null() == true
               if (!resp.is_null()) {
                   // 4. resp.as_array(): [ [stream_name, entries_array] ]
                   auto outer = resp.as_array();
                   for (auto &stream_block : outer) {
                       // 5. Extract the entries list
                       auto entries = stream_block.as_array()[1].as_array();

                       for (auto &entry : entries) {
                           // 6. entry: [id, [field1, val1, ...]]
                           auto rec = entry.as_array();
                           last_id = rec[0].as_string();

                           // 7. Decode key/value pairs
                           auto kvs = rec[1].as_array();
                           std::cout << "Received ID=" << last_id << ", ";
                           for (size_t i = 0; i < kvs.size(); i += 2) {
                               std::cout << kvs[i].as_string()
                                         << "=" << kvs[i+1].as_string() << " ";
                           }
                           std::cout << std::endl;
                       }
                   }
               }
           }
       } catch (const std::exception &e) {
           std::cerr << "Error: " << e.what() << std::endl;
           return EXIT_FAILURE;
       }
       return EXIT_SUCCESS;
   }
   ```

   * **`last_id`**: Tracks the most recent entry ID to avoid re-reading old events.
   * **`XREAD BLOCK 5000 STREAMS event_stream last_id`**: Waits up to 5000 ms for new data after `last_id`.
   * **Parsing**: The Redis reply is a nested array; you iterate to decode each record’s ID and fields.

3. **Compile & Run**

Make sure, to update the filename in your `CMakeLists.txt`-file, then run:

   ```bash
   rm -rf ./build
   cmake -B ./build       
   camke --build ./build  
   ./build/stream_consumer
   ```

4. **Produce Events**

   ```bash
   redis-cli XADD event_stream '*' message "Hello" user "alice"
   ```

   Each event appears in your consumer’s stdout.

---

### Task 3: Publish/Subscribe with C++

**Goal:** Implement a subscriber that listens on `notify_channel` and prints incoming messages, using blocking reads.

1. **`pubsub.cpp` Explanation**

   ```cpp
   #include <iostream>  // std::cout, std::cerr

   #define REDISCPP_HEADER_ONLY
   #include <redis-cpp/stream.h>
   #include <redis-cpp/execute.h>

   int main() {
       try {
           // 1. Connect to Redis
           auto stream = rediscpp::make_stream("localhost", "6379");

           // 2. Subscribe command returns an array confirming the subscription
           rediscpp::execute(*stream, "SUBSCRIBE", "notify_channel");
           std::cout << "Subscribed to notify_channel. Waiting..." << std::endl;

           // 3. Loop forever reading message frames
           while (true) {
               // 4. read_frame() blocks until a new Pub/Sub message arrives
               auto frame = stream->read_frame();
               auto arr = frame.as_array();

               // 5. arr = ["message", channel, payload]
               std::cout << "[" << arr[1].as_string() << "] "
                         << arr[2].as_string() << std::endl;
           }
       } catch (const std::exception &e) {
           std::cerr << "Error: " << e.what() << std::endl;
           return EXIT_FAILURE;
       }
       return EXIT_SUCCESS;
   }
   ```

   * **`SUBSCRIBE`**: Puts the connection into Pub/Sub mode; subsequent calls to `execute` won’t work until you unsubscribe.
   * **`read_frame()`**: Low-level function that fetches the next array frame, blocking until content arrives.
   * **Frame format**: `"message"`, channel name, message payload.

2. **Compile & Run**
Adjust the filename within `CMakeLists.txt` again. 

   ```bash
   rm -rf ./build
   cmake -B ./build       
   camke --build ./build  
   ./build/pubsub
   ```

3. **Publish test messages**

   ```bash
   redis-cli PUBLISH notify_channel "System update at $(date)"
   ```

   Subscriber prints each published payload.

---

## Advice

Working directly with Redis in C++ can feel low-level at first, but by encapsulating each pattern—key-value operations, streams, pub/sub—in its own small application you’ll quickly build muscle memory for the `redis-cpp` API. Notice how the same `make_stream` + `execute` pattern applies across tasks, with only command arguments changing. In production, factor repeated code into utility functions (e.g., `connect()`, `incr_loop()`, `stream_reader()`), and consider asynchronous I/O or thread pools for blocking operations. Keep your CMake setup minimal—once FetchContent brings in **redis-cpp**, you only manage targets and linking. When you expand to pipelines or clustering, revisit this sheet’s structure and adapt the loops into reusable modules. If you enjoyed this exercise, try implementing a rate limiter with Lua scripting or explore Redis Streams consumer groups in another sheet (see [Exercise Sheet: Redis Consumer Groups](https://github.com/STEMgraph/missing)). Good luck and happy coding!
