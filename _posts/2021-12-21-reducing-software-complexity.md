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

I use the GUI library nana for my program. Nana contains code for a message loop already. Nana has an event system that (on windows at least) uses the windows API message system. For example, this is how to specify the action for a button click with nana:

```c++
button.events().click([](const arg_click& arg)
{
	cout << "clicked" << endl;
});
```

Here, click is a function that accepts another function (an event handler) and registers it as an action to be taken when the event (a click) occurs. When the OS sends a message, the program uses some switch or if-else construction to determine the action based on the data in the message. If it's a mouse click at the location of where the button is on the screen, the event handler is called. That's what nana does. The event handler then prints the text "clicked".

When using nana, maintaining consistency after each message translates to maintaining consistency after the execution of each event handler.

Almost everything is an event. Every user action, whether it's a keypress, click, mouse cursor movement... is an event, so responding to user actions can be done entirely by registering event handlers like the one above. There are also other messages such as the WM_PAINT message which is a request from the OS to the program to redraw a part of the window. nana does that kind of stuff automatically. The windows API is low level and more detailed. I only use it for some things that nana can't do.

# Using events to achieve consistency in the GUI

**Child components communicate with parent components through events.**

### component hierarchies

I have been inspired by events in nana and javascript as a reliable method for communication between components. I think the reliability is essentially caused by the principle of parent-child communication. Communication in the direction parent-child is by direct usage of the child component by the parent. The parent may call functions on the child, or change values. Communcation in the direction child-parent is by events. You can think of it like this: parents components are boss. They can do with children what they wish. Children can only raise attention to an issue (say "there is an event!"). The parent decides how to respond.

I think the reason this works well is the same as why a hierarchical power structure works well in organisations of people. It's always clear who/what's responsible: the leader / the parent component. It's a loosely layered approach that distributes program complexity over the many layers, causing the complexity of each individual layer to be manageable. This is also what happens in organisations of people. People have specialized jobs. They do their part. The net result is something so complex that no individual could have realized that. It's only a loosely layered approach because parents can do things with children of children of... etc. and not only their direct children.

In ExploreFractals, the main window is the highest parent component. Its children are the components of the window such as the menubar, tabbar and fractal area, but also child windows. The help window is a child window.

Not everything fits well in the hierarchy model for several reasons: efficiency, my laziness, lack of time and sometimes the model is just not applicable.

### JSON window example

![image](https://user-images.githubusercontent.com/29734312/146872687-ae0a1aa2-2f00-4fe4-b39a-9f46d84c8264.png)

When the apply button in the JSON window is clicked, the following happens, described in natural language (simplified):

1. Windows: to the program: here is a new message
2. Nana: raises event: apply button clicked in the JSON window
3. main_form: to grandgrandchild FractalCanvas: please use these new parameter values
7. FractalCanvas: raises event: the parameters have changed
8. main_form: to grandgrandchild FractalCanvas: start a new render. to grandchild FractalPanel: update the values in the GUI.
10. FractalCanvas: raises event: a new render has been started
11. main_form: starts a refreshthread, to keep the screen up to date during the render
12. FractalCanvas: raises event: the render has finished

You see: the child components only say there is a situation. The parent component responds to the notification with actions and instructions.

This communication may seem cumbersome. The GUI does a request to change the parameters and then receives a notification that the parameters have changed. Of course they have, right? Maybe it is cumbersome, but at least there is one benefit that I see: it makes the code simpler. When the parameters change, the GUI always needs to do something afterwards, like updating the values in the GUI and creating a new history item. Without this event approach, the code to change parameters would look something like this:

```c++
canvas->changeParameters(newParameters, ...);
parameterChangePostActions(...);
```

Everywhere in the GUI code where parameters are changed, those post parameter change actions should not be forgotten. With the event approach, they can never be forgotten. (There is one location in the GUI code where I don't want the post parameter change actions to happen, so I programmed a way to prevent them if needed.)

### events as function calls

In a way, this idea is only a philosophical way to think about what's happening. In reality, there are not separate entities doing things. There is one CPU, with (most of the time) one thread of execution. An event is really a function call, which means that the same CPU now starts doing something else. Delegating work in this sense is like defining the next step for someone else and then doing it yourself.

The way the FractalCanvas raises an event is by calling a function. I have defined some possible events that the GUI can respond to in an interface / abstract class:

```c++
/*
	a (programming) interface for the graphical interface
	Non-GUI classes can request action by the GUI through this interface.

	"Started" means just that: a render has started.
	"Finished" means that the render was 100% completed. Cancelled renders don't ever finish.

	The int "source_id" is an identifier for the cause / source of the event which the GUI can use to handle it.
*/
class GUIInterface {
public:
	virtual void renderStarted(shared_ptr<RenderInterface> render) = 0;
	virtual void renderFinished(shared_ptr<RenderInterface> render) = 0;
	virtual void bitmapRenderStarted(void* canvas, uint bitmapRenderID) = 0;
	virtual void bitmapRenderFinished(void* canvas, uint bitmapRenderID) = 0;
	virtual void parametersChanged(void* canvas, int source_id) = 0;
	virtual void canvasSizeChanged(void* canvas) = 0;
	virtual void canvasResizeFailed(void* canvas, ResizeResult result) = 0;

	virtual ~GUIInterface() {}
};
```

I consider it the responsibility of the FractalCanvas to raise events by calling these functions when appropriate. In the parent-child analogy: if a baby doesn't cry, the parent doesn't know there's something wrong. The baby needs to send a signal.

Here's how the event system helps to achieve consistency in the GUI: if the FractalCanvas code contains no bugs, the GUI will always be notified of parameter changes. If the GUI also contains no bugs, it will always update the values in the GUI when there is a parameter change. (Remark: when I programmed this, changing the values in the GUI also caused a parameter change again, and therefore infinite recursion. I solved this by making use of the source_id.)

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

The idea of program consistency after every (user or non-user) action implies that all actions are handled sequentially. That's because if there can be multiple threads handling actions, at the moment one thread starts handling an action, the program may be in an inconsistent state because of another thread handling an action. This means that threads can make almost no assumptions about the state of the program - it could change any moment because of a different threading doing something. It would be very difficult or even impossible to make this work.

This automatically leads to the idea of using one thread for everything. There are 2 reasons why that's not a good idea:
1. performance. Fractal renders take much less time when multiple cores are used.
2. responsiveness. For the user experience it is crucial that the GUI is resonsive during renders and other operations that can take a long time.

So the next best thing, if one thread for everything is not an option, is to use one thread for _almost_ everything, and extra threads specifically for those situations that require extra threads. The way I now think about it is this: multithreading is used *by exception* only (even if the "exception" of doing renders is the core part of the program). 

As reason (2) indicates, the need for threads depends on how long an operation takes. Almost all event handlers in the program take such a small amount of time to do their work that it's no problem if the program "hangs" during their execution. The time is too short to notice it as a user anyway. Those event handler can all be executed on the main thread - the thread that does almost everything.

The main thread is sometimes called a control thread. It's a common design. I think this design, and the reasons for it, are why games benefit so much from single threaded performance. It's because they use 1 control thread that has way too much to do. In my program it's not a problem. Responding to button clicks etc. is fast enough to be done by 1 thread.

### creating renders and the GUI

When it comes to renders, the following needs to be true:

**The thread that creates new renders for a FractalCanvas should be the thread that destroys the FractalCanvas.**

If not, how can the thread that destroys the FractalCanvas safely do so? The other thread could start new renders any moment. There should not be a thread using the FractalCanvas while it's being destroyed.

This requirement is automatically satisfied by using the main thread for both event handling and creating and changing the GUI. The main thread really does a lot. At the end of the main function, it creates the GUI and then starts handling events. Those events start all actions that are possible in the program, such as closing a tab. When the user closes a tab, the event handler destroys the FractalPanel. Because FractalPanel has a FractalCanvas as one of its class members, the event handler automatically also destroys the FractalCanvas, so the main thread must be the thread that starts renders. It is, because renders are started from event handlers too, as consequences of actions.

### render progress and a responsive GUI

There are 2 situations where I have chosen to use new threads, other than for rendering the fractal:

1. showing progress
2. cancelling renders

By showing progress I don't mean to just show the percentage of how much is done, but also to show the calculated pixels on the screen. That's very important for the user experience. ExploreFractals updates the screen 10 times per second during fractal calculations. Meanwhile it must be possible to use the GUI. It's not user friendly to be forced to wait for a render to finish completely before being able to do something else.

Cancelling renders can take time, because there are threads that are working that need to stop. I think there's room for improvement there, but right now it takes time. To keep the GUI responsive during the cancellation, it also needs to be done in a different thread.

### challenges with multiple threads

The challenge is complexity. There is so much going on in a program that it's very easy to forget some rare situation that can happen. You need to constantly ask yourself questions like "can it POSSIBLY be true that a refreshthread is executing _while_ a tab is being closed?" Problems occur when two special situations are handled well in isolation, but not when they occur both at the same time. Threads execute at the same time, so you get a lot of those "at the same time" problems. Some things that should not happen at the same time:

- When a tab is closed during a render, its memory should not be freed until all renders, recolorings and progress updates are finished.
- A new render is started at (almost) exactly the same time a previous render finished. Both threads update the number of renders in the queue with a new value at the same time (a race condition).

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
	
Preventing race conditions can be done with mutex locks; problem solved.

The other challenge is to ensure that a FractalCanvas is only destroyed after all threads using it are finished. The solution I use is counting the number of threads, and freeing the resource only when there are 0 threads using it. The program consistency after each message can be used to guarantee that no new threads will start using the resource after that.
	
For example, when a refreshthread is started (which refreshes the screen 10 times per second), a variable in the FractalCanvas which counts the number of non-render threads is updated, and again updated when the thread ends:

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

This is only to count the number of non-render threads. I also use the (FractalCanvas) variables renderQueueSize and bitmapRenderQueueSize to count the number of renders and recolorings (in the code called bitmap renders) in the queue respectively.
 
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

These loops make sure that all threads using the FractalCanvas are finished. The first while loop keeps cancelling renders until the queue is empty. It requests a lock for mutex activeRender. If that succeeds, there can be no active render, which ensures that renderQueueSize == 0 indeed means there are no more renders, not even an active one.

The second while loop does the same for bitmap renders, and in addition to that it also checks that the number of other threads is 0. This number should go to 0 automatically when all renders are finished because refreshthreads terminate themselves when the render is finished. The loop keeps spinning until all threads are finished.

This only works if no new renders are added to the queue, which is ensured by the way messages are handled. As long as a FractalCanvas exists in a tab, a user can cause new renders to start by changing the parameters, which is fine. Once the user closes the tab, the tab close event handler starts executing, which starts the cleanup of the FractalCanvas, which is the destructor code above. The event handler also removes the tab from the GUI. By the time the event is over, and the next message is received, the tab does not exist anymore, and another tab (or no tab at all) will have focus. Even if the next message is a button click from the user that would change the parameters, that click will affect the newly focused tab and not the old one. It's impossible for the user to cause new renders for the FractalCanvas in the closed tab. This works because messages are sequentially handled by one single thread.

# Simple design

All of the above is part of a more general idea: simple design makes bugs less likely. The control thread design is simple because there can be no doubt about which thread is executing the event handlers. It's always the same thread.

### simple event handlers

The way I programmed event handlers is also simple. There are 2 principles behind the event handlers, the first of which I already mentioned:

1. The pre-condition and post-condition are that the program is in a consistent state.
2. There can't be any assumptions about what the previous or next message is

The second principe is there out of necessity, because the windows API message system doesn't offer any guarantees about the order of messages, but it's also simplifying. It rules out any logic *depending* on the order of messages. If there were such logic, that would automatically make the program more complex. Another benefit is that code with fewer dependencies is easier to re-use.

### simplification out of necessity or as a precaution

I started thinking about simplification out of necessity. There were bugs in the program and I found it really difficult to fix them. This made me think: if simplification helps when the complexity is overwhelming, maybe it also helps when the complexity is already manageable. I now think all the time: how can I make this simpler? That is without sacrificing performance and user experience. I already think the function above is a bit complex.
	
# Encapsulating complexity
	
### the render queue
	
Sometimes there is no way to avoid complexity. An example of this is the render queues for the FractalCanvas. I really needed those queues for responsiveness, so I had to implement them. The implementation is encapsulated in a function.
	
```c++
//
// There can be multiple threads waiting to start if the user scrolls very fast and many renders per second are started and have to be cancelled again. To deal with that situation, this function places all new renders in a queue by using the mutex activeRender.
// In addition, the mutex renderInfo is used to protect the variables that keep information about the number of renders, the last render ID etc.
//
void enqueueRender(bool new_thread = true)
{
	auto action = [this](uint renderID)
	{
		{
			lock_guard<mutex> guard(activeRender);
			//if here, it's this thread's turn
			{
				lock_guard<mutex> guard(genericMutex);
				bool newerRenderExists = lastRenderID > renderID;

				if (newerRenderExists)
				{
					if(debug) cout << "not starting render " << renderID << " as there's a newer render with ID " << lastRenderID << endl;
					renderQueueSize--;
					return;
				}
				else {
					activeRenders++;
				}
			}
			if(debug) cout << "starting render with ID " << renderID << endl;
			createNewRender(renderID);
			{
				lock_guard<mutex> guard(genericMutex);
				activeRenders--;
				renderQueueSize--;
			}
		}
		//After releasing the lock on activeRender, nothing should be done that uses the FractalCanvas. When the user closes a tab during a render, the FractalCanvas gets destroyed immediately after the lock is released.
	};

	uint renderID;

	//update values to reflect that there's a new render
	{
		lock_guard<mutex> guard(genericMutex);
		renderQueueSize++;
		renderID = ++lastRenderID;
	}

	//actually start the render
	if (new_thread)
		thread(action, renderID).detach();
	else
		action(renderID);
}
```
	
It took time to get the logic right, but eventually it worked and the problem was solved. In the GUI code, all that's needed to start a new render is calling that function, which makes starting a render really simple and therefore less prone to bugs.
	
### parameter changes
	
Another example of this is parameter changes. Every time there is a parameter change, logic is needed to determine whether the change requires a recalculation, a recoloring or a memory allocation. That's important because changing only the colors should not cause a whole new render of the fractal, for example. That wouldn't be good for the user experience.

I was inspired by how nana provides functions that accept other functions. I created the function FractalCanvas::changeParameters that accepts any parameter changing function:
	
```c++
ResizeResult changeParameters(std::function<void(FractalParameters&)> action, int source_id = 0, bool check_modified_memory = true)
{
	...
```
	
changeParameters is not very efficient, because it first applies the supplied action to a temporary copy of the current FractalParameters object, just to see if the action _would_ require a reallocation of memory, a recalculation etc. This doesn't matter because it's still very fast, and the benefit of having changeParameters is great. In the GUI code, changing the parameters is as simple as doing something like this (this is the behavior of the toggle julia button):
	
```c++
canvas->changeParameters([](FractalParameters& P)
{
	P.setJulia( ! P.get_julia());
	if (P.get_julia()) {
		P.setJuliaSeed( P.map_transformations(P.get_center()) );
		P.setCenterAndZoomAbsolute(0, 0);
	}
	else {
		P.setCenterAndZoomAbsolute(P.get_juliaSeed(), 0);
	}
}, EventSource::julia);
```
	
changeParameters knows nothing about the kind of parameter changing function it receives. It just applies the function and then investigates what it needs to do based on its effects, for example: allocate new memory if the size has changed, wait for existing renders if so, and raise a parameter changed event.
	
I think this example illustrates the benefit of encapsulation well. Using changeParameters makes it impossible to forget any of that, as long as the implementation of changeParameters is correct. A reoccuring danger of bugs is reduced to making only one part of the program bug free. With this approach, it either goes wrong everywhere or nowhere.
