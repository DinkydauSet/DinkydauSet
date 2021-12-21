---
layout: post
title:  "Reducing software complexity by design in ExploreFractals"
---

While I was working on my program ExploreFractals I had segfaults, deadlocks, memory leaks, inconsistencies etc. I had to find solutions for these problems. What I eventually came up with are some ideas that all work together.

# Keeping the program in a consistent state

**I want the program to be in a consistent state after every action.** For example, when the user clicks a button, the program may be temporarily in an inconsistent state during the handling of the click, but it should be consistent again before the next action is handled. This makes it possible to do assumptions. When writing code that handles a click, I know "The program is in a consistent state when this code begins, so..." and then various conclusions. Being able to make those assumptions makes writing the code a lot easier.

On a lower level, actions correspond to messages from the OS (which is windows - my program is for windows (and works in wine)). This is how windows is designed. So what I want is the program to be consistent after every message.

<div style="background-color: rgb(247, 242, 232);">

### messages

The OS can send messages to a program. The program can choose to handle them. If not, they just pile up, like a mailbox that becomes fuller and fuller. The common way to handle messages is with a message loop, like this:

```c++
MSG msg;
while (GetMessage(&msg, NULL, 0, 0)) {
	TranslateMessage(&msg);
	DispatchMessage(&msg);
}
```

This shows that the program is in control of what it's doing, not the OS. The program *asks* the OS for a new message (GetMessage). If there is one, action can be taken depending on the data in the message object.  DispatchMessage calls the message handling function of the window that the message is indended for. Here you see that there is one message loop for the whole program, even if it has multiple windows. 

Functions like GetMessage are windows API functions. The functions are declared in C++ so that they can be used in code, but they are not defined in C++. The definition is in compiled libraries from windows that the program can be linked to. What the functions do is transfer control to the OS by using a CPU instruction. That means that by calling a windows API function, the OS is (temporarily) in control of what the program does. (The OS is always in control over *if* a program is executing or not.)

Here's a way the OS uses that control: the call to GetMessage only finishes when there is a message. The call is said to be "blocking". This is what causes a GUI program to use (near) 0% of the CPU when the user does nothing. The program is always either working or waiting for a new message. This is noteworthy because even though programs that can be "idle", such as GUIs, are very common, it's not the nature of a program to do nothing. C++ code specifies actions to be done, in sequence, until the program terminates. There is no code to "do nothing".
	
</div>

### nana GUI library

I use the GUI library nana for my program. Nana contains code for such a message loop already. Nana has an event system that (on windows at least) uses the windows API message system. For example, this is how to specify the action for a button click with nana:

```c++
button.events().click([](const arg_click& arg)
{
	cout << "clicked" << endl;
});
```

Here, click is a function that accepts another function (an event handler) and registers it as an action to be taken when the event (a click) occurs. When the OS sends a message, the program uses some switch or if-else construction to determine the action based on the data in the message. If it's a mouse click at the location of where the button is on the screen, the event handler is called. That's what nana does. The event handler then prints the text "clicked".

When using nana, maintaining consistency after each message translates to maintaining consistency after the execution of each event handler.

Almost everything is an event. Every user action, whether it's a keypress, click, mouse cursor movement... is an event, so responding to user actions can be done entirely by registering event handlers like the one above. There are also other messages such as the WM_PAINT message which is a request from the OS to the program to redraw a part of the window. nana does that kind of stuff automatically. The windows API is low level and more detailed. I only use it for some things that nana can't do.

# One thread for everything

**I want one thread that does everything there is to do, with exceptions to that rule only when strictly necessary.**

<div style="background-color: rgb(247, 242, 232);">
	
### threads of execution

If you think of a program as code that is being executed, one instruction after the other, that's a single threaded program. Only one instruction is being executed at any one time. A multithreaded program is a program where multiple parts of the code are executing at the same time. Each execution is a thread of execution. Having multiple parts of the code executing at the same time is literally possible if the CPU has multiple cores, but operating systems also simulate multithreading, so you don't need as many cores as there are threads for the observable behavior of a multithreaded program. The benefit of multiple CPU cores is just that the computer is more responsive and gets more work done because there are actually multiple cores working.

A thread can be created in c++ like this:

```c++
thread([]()
{
	//This is done in the new thread:
	cout << "this is printed from a different thread" << endl;
	//When here, the thread ends
}).detach();
```

"thread" is a constructor that takes a function that the thread should execute. The "detach" means that the thread is detached from the constructed thread object, which in practice means that the thread will keep executing until it's done, even if the thread object goes out of scope.
	
</div>

### using one thread for everything

Multiple threads can be problematic. If multiple messages are being handled at the same time by different threads, the handling of one message can put the program in an inconsistent state. The event handler for the other message doesn't know that, so in order for multithreaded event handling to work, event handlers can make almost no assumptions about the state of the program, which makes it difficult to do anything.

This makes it attractive to use only one thread for the entire program, but that's not practical. My program renders fractals, which requires many computations. A compromise is to do fractal calculations with multiple threads, so that multiple CPU cores are used, and do everything else in one specific thread. The way I now think about it is this: multithreading is used *by exception* only, even if the "exception" is the core part of the program.

The thread that does almost everything is sometimes called a control thread. It's a common design. I think this design, and the reasons for it, are why games benefit so much from single threaded performance. It's because they use 1 control thread that has way too much to do. In my program it's not a problem. Responding to button clicks etc. is fast enough to be done by 1 thread.

### render progress and a responsive GUI

There are 2 situations where I have chosen to use new threads, other than for rendering the fractal:

1. showing progress
2. cancelling renders

By showing progress I don't mean to just show the percentage of how much is done, but also to show the calculated pixels on the screen. That's very important for the user experience. ExploreFractals updates the screen 10 times per second during fractal calculations. Meanwhile it must be possible to use the GUI. It's not user friendly to be forced to wait for a render to finish completely before being able to do something else.

Cancelling renders can take time, because there are threads that are working that need to stop. I think there's room for improvement there, but right now it takes time. To keep the GUI responsive during the cancellation, it also needs to be done in a different thread.

### challenges with multiple threads

The reason I use multithreading only when really necessary is that it's difficult to do without bugs. The challenge is complexity. There is so much going on in a program that it's very easy to forget some rare situation that can happen. You need to constantly ask yourself questions like "can it POSSIBLY be true that a refreshthread is executing while a tab is being closed?" Problems occur when two special situations are handled well in isolation, but not when they occur both at the same time. Threads execute at the same time, so you get a lot of those "at the same time" problems. Some things that should not happen at the same time:

- When a tab is closed during a render, its memory should not be freed until all renders, gradient changes and progress updates are finished.
- A new render is started at almost exactly the same time a previous render finished. Both threads update the number of renders in the queue with a new value (a race condition).

The fact that there is a queue of renders at all is because of my requirement to keep the GUI responsive. When the user changes parameters very quickly (for example by using the gradient slider) many recolorings of the fractal are started. I want the slider to remain responsive and thus able to start new recolorings while a recoloring is still taking place. The new pending recoloring should start when the previous recoloring has ended. You can see that this requirement creates a new problem, because the existence of the queue means that when a tab is closed, waiting for the current render to finish before freeing the resources is not enough. Also, it's important to guarantee that even with a responsive GUI at hand, the user can't add more renders and recolorings to the queue after the tab has closed. Otherwise, the queue may become empty, the resources start to be freed because the queue has become empty, and then another renders starts, using the freed resources, resulting in undefined behavior.

Multithreading is complex. That's why having a single control thread is appealing. Any simplifying design is appealing.

<div style="background-color: rgb(247, 242, 232);">

### mutex locks

The second point above is about a race condition. The behavior of the program depends on the timing of the execution of 2 threads. A race condition is not always a problem. It's only a problem if one of the possible outcomes of the race is bad. In this case, one of the outcomes is indeed bad. The intended sequence of actions is either

> thread 1: render is finished  
> thread 1: reads the number of renders in the queue  
> thread 1: change the value to (old_value - 1)  
> thread 2: new render starts  
> thread 2: reads the number of renders in the queue  
> thread 2: change the value to (old_value + 1)

or 
>
> thread 2: new render starts  
> thread 2: reads the number of renders in the queue  
> thread 2: change the value to (old_value + 1)  
> thread 1: render is finished  
> thread 1: reads the number of renders in the queue  
> thread 1: change the value to (old_value - 1)
>
If the threads are performing their actions at almost exactly the same time, the actions can happen in an intertwined order such as:
>
> thread 2: new render starts  
> thread 1: render is finished  
> thread 2: reads the number of renders in the queue  
> thread 1: reads the number of renders in the queue  
> thread 2: change the value to (old_value + 1)  
> thread 1: change the value to (old_value - 1)

Now the number of renders in the queue is incorrect, because at the time thread 1 writes its change, the old value it's using was already changed again. Note that the cause of the problem has something to do with the fact that adding to a value in memory consists of more than 1 "atomic" operations: a read, an addition and a write.

This problem can be solved with a mechanism for mutually exclusive access, which exists in C++:

```c++
mutex m;
void testfunction() {
	lock_guard<mutex> guard(m);
	//only 1 thread can execute the remaining code in this block at the same time
	some_shared_variable += 1; //example protected action
}
```

The same mutex can be used in multiple blocks of code, in which case only 1 thread is allowed to execute code in any of those blocks. More precisely: only one `lock_guard<mutex>` with the same mutex m can exist at any one time. If a thread tries to create one, while another one already exists, it waits. Mutexes can be used to protect pieces of code that change a value that is being used by multiple threads. It's also a little dangerous, because a deadlock situation can occur when 2 threads are both waiting for a mutex that the other thread is using. When that happens, both threads will wait forever.

</div>

### solution for the challenges

The solution for the resource freeing problem that I use: count the number of threads using the resource, and free it only when there are 0 threads using it. Use program consistency after each message to guarantee that no new threads will start using the resource after that. For example, when a refreshthread is started (which refreshes the screen 10 times per second), a variable in the FractalCanvas which counts the number of non-render threads is updated, and again updated when the thread ends:

```c++
//start a refresh thread
canvas->addToThreadcount(1);
thread refreshThread([=]()
{
	refreshDuringRender(std::move(render), canvas, renderID);
	canvas->addToThreadcount(-1);
});
refreshThread.detach();
```

Because there is a risk that multiple threads update the threadcount at the same time, I use a mutex lock in the update function:

```c++
void addToThreadcount(int amount) {
	lock_guard<mutex> guard(genericMutex);
	assert((int64)otherActiveThreads + amount >= 0); //there should not be less than 0 threads
	otherActiveThreads += amount;
}
```

It's important to count the number of other/non-render threads carefully. When a tab is closed and the FractalCanvas needs to be deleted from memory, there may be multiple renders and a refreshthread busy. The number of renders and recolorings (in the code called bitmap renders) in the queue is counted by renderQueueSize and bitmapRenderQueueSize respectively.
 
In the destructor of FractalCanvas I do

```c++
while (true) {
	cancelRender();
	lock_guard<mutex> guard(activeRender);
	if (renderQueueSize == 0)
		break;
}

while (true) {
	cancelBitmapRender();
	lock_guard<mutex> guard(activeBitmapRender);
	if (bitmapRenderQueueSize == 0 && otherActiveThreads == 0)
		break;
}
```

The first while loop keeps cancelling renders until the queue is empty. It requests a lock for mutex activeRender. If that succeeds, there can be no active render, which ensures that renderQueueSize == 0 indeed means there are no more renders, not even an active one.

The second while loop does the same for bitmap renders, and in addition to that it also checks that the number of other threads is 0. This number should go to 0 automatically when all renders are finished because refreshthreads terminate themselves when the render is finished. The loop keeps spinning until all threads are finished.

This only works if no new renders are added to the queue, which is ensured by the way messages are handled. As long as a FractalCanvas exists in a tab, a user can cause new renders to start by changing the parameters, which is fine. Once the user closes the tab, the tab close event handler starts executing, which starts the cleanup of the FractalCanvas, which is the destructor code above. The event handler also removes the tab from the GUI. By the time the event is over, and the next message is received, the tab does not exist anymore, and another tab (or no tab at all) will have focus. Even if the next message is a button click from the user that would change the parameters, that click will affect the newly focused tab and not the old one. It's impossible for the user to cause new renders for the FractalCanvas in the closed tab. This works because messages are sequentially handled by one single thread.

## Simple design

All of the above is part of a more general idea: simple design makes bugs less likely. The control thread design is simple because there can be no doubt about which thread is executing the event handlers. It's always the same thread.

### simple event handlers

The way I programmed event handlers is also simple. There are 2 principles behind the event handlers, the first of which I already mentioned:

1. The pre-condition and post-condition are that the program is in a consistent state.
2. There can't be any assumptions about what the previous or next message is

The second principe is there out of necessity, because the windows API message system doesn't offer any guarantees about the order of messages, but it's also simplifying. It rules out any logic *depending* on the order of messages. If there were such logic, that would automatically make the program more complex. Another benefit is that code with fewer dependencies is easier to re-use.

### simplification out of necessity or as a precaution

I started thinking about simplification out of necessity. There were bugs in the program and I found it really difficult to fix them. This made me think: if simplification helps when the complexity is overwhelming, maybe it also helps when the complexity is already manageable. I now think all the time: how can I make this simpler? That is without sacrificing performance and user experience.
