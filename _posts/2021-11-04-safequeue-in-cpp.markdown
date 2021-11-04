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


The full source is here:

<script src="https://gist.github.com/madmax440/7e0f129e810a310e86ea4e4a9d5dc2c6.js"></script>

<script src="https://gist.github.com/madmax440/c233a436d449a61008eb87ff26005474.js"></script>
