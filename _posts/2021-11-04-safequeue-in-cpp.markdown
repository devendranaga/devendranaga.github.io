---
layout: post
title:  "safequeue implementation in c++"
date:   2021-11-04 01:04:00
categories: cryptography, c++
---

TL;DR, this is a post with regards to the challenges few engineers (may be many) have faced when passing messages between classes and threads in C++.
This made me think of a common library API interface may be even can be simplest yet more useful can be done.

The design is as follows,

1. A queue that can publish to multiple subscribers from many publishers.
2. A thread notify subscribers immediately soon as publisher generates messages.
3. A queue to store subscriber callbacks, which will be called later in the program.
4. support for any types - use of templates

A usecase of this is sensor data processing. A camera / lidar receive thread will listen on the interface and read packets as and when they are received. A processor thread(s) will run on the received data to perform further actions on the data.

The class will then be created the following way, in C++

```cpp

namespace Auto_OS::Lib {

template <typename T>
struct Subscriber_Info {
    // id of the subscriber - for informational purposes
    int sub_id;

    // receive callback to bve called by the notifier thread
    std::function<void(int, T)> receive_callback;
};

template <typename T>
class SafeQueue {
    public:
        ~SafeQueue() { }

        explicit SafeQueue(const SafeQueue &) = delete;
        const SafeQueue &operator=(const SafeQueue &) = delete;
        explicit SafeQueue(const SafeQueue &&) = delete;
        const SafeQueue &&operator=(const SafeQueue &&) = delete;

        static SafeQueue *Instance()
        {
            static SafeQueue q;
            return &q;
        }

        void QueueItem(T &item);

        void Subscribe(int sub_id, std::function<void(int, T)> rx_cb);

        int Unsubscribe(int id);

    private:
        explicit SafeQueue();
        std::mutex lock_;
        std::condition_variable cond_;
        std::queue<T> items_;
        std::unique_ptr<std::thread> dispatch_thr_;
        std::vector<Subscriber_Info<T>> subscribers_;

        void Dispatcher();
};

```

So Now there is a way to 

1. Queue items as and when the publisher produces them - `QueueItem`.
2. Subscribe interface to setup callbacks so the SafeQueue will dispatch when there is items to read - `Subscribe`.
3. Unsubscribe to the events when not needed - `Unsubscribe`.
4. Dispatcher that will call Subscribed callbacks - `Dispatcher`.


Lets say that if we want to queue a type into the Safequeue, we can do this with the following call. For example, a one line call to store an integer type can be done as follow.

```cpp
int data = 1;

SafeQueue<int>::Instance()->QueueItem(data);
```

Now to receive the item, one can simply subscribe only once on the data type. The callbacks will be called automatically by the dispatcher.

```cpp
int sub_id = 1;

SafeQueue<int>::Instance()->Subscribe(sub_id, OnDataReceive);

...

void OnDataReceive(int sub_id, int data)
{
    printf("subscriber[%d] received data: %d\n", sub_id, data);
}

```

The caller of this function will be the `Dispatcher` thread that is instantiated when the first instance of the type is created with `Instance` member. By this call,

```cpp
instance = SafeQueue<int>::Instance();
```

this will create a thread for the integer types, since the template code generated at compile time, if there are many types, then there will be many such instances of SafeQueue objects handling each type.


The full source is here:

The SafeQueue is implemented in a simple header file here:

<script src="https://gist.github.com/madmax440/7e0f129e810a310e86ea4e4a9d5dc2c6.js"></script>

To test the SafeQueue i created a sample program, containing two threads, one produces values and other consumes. So to validate multi publisher and subscriber, i created two publishers one doing integers counting upwards, and one publishing strings in periodic intervals. There are 6 subscribers for integer publisher and two subscribers for the string publisher.

Remember that the calling order of the subscribers follows the order they were being registered. So if the subscriber need to be called early, it need to be registered earlier.

Since the calling is sequence, the callbacks shouldn't do anymore work than usual, may be the callback should just copy the value and interrupt a thread to process the data. If the callback takes up time, the next callback calls will be delayed thus adding latency. There is no way to resolve this, even not possible with multi-threads or thread-pool based dispatchers.

The sample code test is as follows,

<script src="https://gist.github.com/madmax440/c233a436d449a61008eb87ff26005474.js"></script>
