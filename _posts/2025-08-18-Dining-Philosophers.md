---
layout: post
title: The Dining Philosophers
date: 2025-08-18 21:00:00
description: Understanding Djikstra's solution to the dining philosophers problem.
tags: code learning developer
categories: tech
---

# The Problem

This is a classical computer science concurrency problem with limited resources. The problem statement is as follows: imagine 5 philosophers dining at a round table, each with their own plate of spaghetti in front of them. There are only 5 forks, one between each philosopher. A philosopher can be in one of two states: thinking or eating. They are only allowed to eat when they have both the left and right fork (the limited resource). A key constraint is that they can only pick up one fork at a time, never both simultaneously. They also cannot attempt to pick up a fork that is already in use by their neighbor. Once they are done eating, both forks are released. Assuming the philosophers cannot communicate when they start or finish eating, how do you design a procedure for the philosophers to eat without any one of them starving?

{% include figure.liquid loading="eager" path="assets/img/diningphil.png" class="img-fluid rounded z-depth-1" zoomable=true %}

<div class="caption">
    A Netflix show called Pantheon I was watching featured the dining philosophers problem, with chopsticks and tofu stir fry instead of the usual forks and spaghetti.
</div>

In Operating Systems speak, the philosophers are processes, and the forks are limited resources that must be shared in synchronized fashion!

An issue you want to avoid is ending up in deadlock, where you are stuck in a circular dependency. For example if the procedure requires the philosopher to first try to pick up the fork on their left and then wait for the right fork to be available, it is possible that each philosopher takes the fork to their left, and sits there waiting for the fork on their right to be set down. However, since the fork on the right is being held by a philosopher who is also waiting for the next fork, the situation cannot progress and all the philosophers will starve.

# Parity Solution

One easy solution to avoid deadlock is to use parity of the philosophers to ensure that this circular waiting does not occur. Numbering philosophers one through five, odd philosophers first try to pick up the fork on their left, while even philosophers first try to pick up the fork on their right. This way when a philosopher acquires a fork, the other fork it waits on can only be the "second" fork for the neighbor and never the first fork, in which case it is guaranteed to be set down and available at some future point in time. For instance, if the even philosopher picks up the right fork and waits on the left fork there are two possibilities: either that left fork is available and he succeeds, or it is held by an odd philosopher who has acquired both forks and is currently eating, and will soon enough put the fork down. This solution is obviously asymmetric as there will be philosophers 1 and 5 sat next to each other, and is more effective when you have an even number of philosophers and forks.

# Hierarchical Solution

Another solution is to give an order to the forks or resources. Numbering the forks 1 through 5, set a rule in which each philosopher must first acquire the lower numbered fork before attempting to acquire the higher numbered one. In the case that the rank of the forks increases clockwise, 4 of the philosophers will have the lower numbered fork to their right and will be "right-handed" while one philosopher with the 1 fork to the left and the 5 fork to their right will be "left-handed". This circumvents the deadlock by preventing circular waiting. If each philosopher simultaneously obtains their lower numbered fork, only four can succeed, with one of the four philosophers (i.e. with 5 fork on left and 4 fork on right) free to acquire the highest numbered fork as their second and proceed.

# Waiter Solution

A more straightforward and obvious solution is to put the action of checking/changing fork states (free or in-use) inside a critical section. The analogy is that the philosopher must obtain permission to eat from a "waiter". When requesting permission, the waiter first checks the availability of the two forks if they are free. If so, the waiter marks them as in-use, and gives permission to the philosopher to eat. In practice, the waiter is represented as a mutex lock, which once acquired, checks fork states. The mutex is released if one of the two forks is not free. If both forks are free, their states are changed to in-use before the lock is released (in which case the philosopher has permission to pick them up). This way, we make checking and picking up two forks atomic rather than sequential. This prevents any philosopher from being caught waiting for the second fork while holding the first, or for a neighbor to take a fork before the philosopher tries to pick it up. This can have good parallelism as failing to acquire both forks (permission denied) does not prevent other philosophers from subsequently acquiring forks.

# Djikstra's Solution

The solution I found really interesting was the one proposed by Djikstra. It uses both mutexes and semaphores, and abstracts away individual forks. Instead of looking at the problem from the perspective of forks, Djikstra looks from the perspective of whether the neighboring philosophers are eating (if neighbors are not eating, then you can eat). The following is a C++ implementation which I have yanked from Wikipedia:

```c++
#include <chrono>
#include <iostream>
#include <mutex>
#include <random>
#include <semaphore>
#include <thread>

constexpr const size_t N = 5;  // number of philosophers (and forks)
enum class State
{
    THINKING = 0,  // philosopher is THINKING
    HUNGRY = 1,    // philosopher is trying to get forks
    EATING = 2,    // philosopher is EATING
};

size_t inline left(size_t i)
{
    // number of the left neighbor of philosopher i, for whom both forks are available
    return (i - 1 + N) % N; // N is added for the case when  i - 1 is negative
}

size_t inline right(size_t i)
{
    // number of the right neighbor of the philosopher i, for whom both forks are available
    return (i + 1) % N;
}

State state[N];  // array to keep track of everyone's both_forks_available state

std::mutex critical_region_mtx;  // mutual exclusion for critical regions for
// (picking up and putting down the forks)
std::mutex output_mtx;  // for synchronized cout (printing THINKING/HUNGRY/EATING status)

// array of binary semaphores, one semaphore per philosopher.
// Acquired semaphore means philosopher i has acquired (blocked) two forks
std::binary_semaphore both_forks_available[N]
{
    std::binary_semaphore{0}, std::binary_semaphore{0},
    std::binary_semaphore{0}, std::binary_semaphore{0},
    std::binary_semaphore{0}
};

size_t my_rand(size_t min, size_t max)
{
    static std::mt19937 rnd(std::time(nullptr));
    return std::uniform_int_distribution<>(min, max)(rnd);
}

void test(size_t i)
// if philosopher i is hungry and both neighbors are not eating then eat
{
    // i: philosopher number, from 0 to N-1
    if (state[i] == State::HUNGRY &&
        state[left(i)] != State::EATING &&
        state[right(i)] != State::EATING)
    {
        state[i] = State::EATING;
        both_forks_available[i].release(); // forks are no longer needed for this eat session
    }
}

void think(size_t i)
{
    size_t duration = my_rand(400, 800);
    {
        std::lock_guard<std::mutex> lk(output_mtx); // critical section for uninterrupted print
        std::cout << i << " is thinking " << duration << "ms\n";
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(duration));
}

void take_forks(size_t i)
{
    {
        std::lock_guard<std::mutex> lk{critical_region_mtx};  // enter critical region
        state[i] = State::HUNGRY;  // record fact that philosopher i is State::HUNGRY
        {
            std::lock_guard<std::mutex> lk(output_mtx); // critical section for uninterrupted print
            std::cout << "\t\t" << i << " is State::HUNGRY\n";
        }
        test(i);                        // try to acquire (a permit for) 2 forks
    }                                   // exit critical region
    both_forks_available[i].acquire();  // block if forks were not acquired
}

void eat(size_t i)
{
    size_t duration = my_rand(400, 800);
    {
        std::lock_guard<std::mutex> lk(output_mtx); // critical section for uninterrupted print
        std::cout << "\t\t\t\t" << i << " is eating " << duration << "ms\n";
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(duration));
}

void put_forks(size_t i)
{

    std::lock_guard<std::mutex> lk{critical_region_mtx};    // enter critical region
    state[i] = State::THINKING;  // philosopher has finished State::EATING
    test(left(i));               // see if left neighbor can now eat
    test(right(i));              // see if right neighbor can now eat
                                 // exit critical region by exiting the function
}

void philosopher(size_t i)
{
    while (true)
    {                         // repeat forever
        think(i);             // philosopher is State::THINKING
        take_forks(i);        // acquire two forks or block
        eat(i);               // yum-yum, spaghetti
        put_forks(i);         // put both forks back on table and check if neighbors can eat
    }
}

int main() {
    std::cout << "dp_14\n";

    std::jthread t0([&] { philosopher(0); }); // [&] means every variable outside the ensuing lambda
    std::jthread t1([&] { philosopher(1); }); // is captured by reference
    std::jthread t2([&] { philosopher(2); });
    std::jthread t3([&] { philosopher(3); });
    std::jthread t4([&] { philosopher(4); });
}
```

Let's break this solution down part by part. First thing to observe is that instead of assigning states to forks, Djikstra assigns 3 states to philosophers: `THINKING`, `HUNGRY`, and `EATING`. We have a mutex lock `critical_region_mtx` that protects checking philosopher states, transitioning states, and calling a special permission function `test`. There is also a binary semaphore for each philosopher in `both_forks_available` that represents both forks being available as a singular resource (`1` meaning both forks available, `0` meaning at least one fork is in use by a neighbor). The `both_forks_available` semaphores are initialized to `0` so no forks are available in the beginning (you will see why this is important soon).

When a philosopher tries to eat, he must first enter the critical region to change to the `HUNGRY` state and check with `test` whether the left and right philosophers are eating. The `test` if successful (both neighbors not eating), releases the `both_forks_available` binary semaphore, essentially signaling that the two forks are now available to be picked up (permission given to self!). We see now that the reason the binary semaphores are initialized to `0` is because it requires the philosophers to first check the neighbor states before receiving permission. Once the `test` is conducted, the philosopher then can exit the critical region and attempt to acquire the forks by decrementing the binary semaphore. If `test` previously granted permission, this will succeed, otherwise the philosopher will be blocked.

Now let's look at what happens next with the two possibilities (permission granted or blocked):

### Permission granted:

With the binary semaphore acquired, the philosopher has permission to pick up the forks without worrying about them being taken and can eat at his own pace (his `EATING` state bars neighbors from trying). When finished eating, the philosopher must put the fork down by entering the critical region to change to the `THINKING` state and then call `test` on behalf of the left and right neighbors. This is an important step because once the forks are freed, the left and right neighbors who may be blocked can then be released and given permission to eat.

### Blocked:

If the philosopher is blocked, this means that one of the neighbors was in the `EATING` state when the `test` was conducted. However, it is guaranteed not to block forever because the neighbor once finished eating will call `test` on the blocked philosophers behalf. If both neighbors are not `EATING` at this point, the blocked philosopher is released and acquires the `both_forks_available` binary semaphore, gaining permission to eat!

I do like this solution because while complicated, it is symmetric and guaranteed to be fair. If one of the neighbors is eating at the time of the `test` attempt, the philosopher can safely block on the binary semaphore and know that when the neighbor is finished, he will be notified and immediately given permission to eat.

# Mutex Locks vs. Binary Semaphores

I think it is important to mention the distinction of mutexes and semaphores. When I initially learned about them, I thought locks and binary semaphores were the same. But there are fundamental differences in how they operate. With binary semaphores, anyone can call `signal()` on the semaphore and increment the semaphore to 1. Whereas with locks, the only person who can release it is the person who acquired the mutex. In short: a mutex has clear **ownership property**, while binary semaphores are **agnostic to ownership** and can be signalled by any thread.

Binary semaphores are useful for signaling and coordination between threads (e.g. producer-consumer relationships) while mutexes are good for protecting critical sections.
