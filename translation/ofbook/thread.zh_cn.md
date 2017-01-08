# Threads

*by [Arturo Castro](http://arturocastro.net)*

*corrections by Brannon Dorsey*

*translate by ryanaltair*

## 什么是线程(thread)和什么时候需要使用它
 
在一个应用程序中，我们可能会碰到需要耗费些许时间的任务。例如，从硬盘上读取一些东西，CPU的读取内存的速度总是快于硬盘，那么，需要读取一张保存在硬盘中的高清图片，对于应用程序的其他任务来说，就是一个长时间的任务了。
 
在openFrameworks， 总是需要与openGL协同工作，我们的应用程序需要不断进行重复update/draw这个循环。
如果打开了垂直同步(vertical sync)，那么我们的屏幕会保存60Hz的刷新率，那么这个循环的执行时间就只有16ms(1s/(60frames/s))*1000(ms/s).
而从硬盘读取一张图片总是会超过16ms的，因此，如果在update中读取图片，那么，我们会注意到程序假死了。
 
  
为了解决这个问题，我们使用线程(threads)。线程指在主程序工作流外执行的一个特定任务。可以让我们可以一次同时执行多个任务而不会令我们的主程序假死。我们也可以将大型任务分开，变成多个小块的任务，加快处理速度。你可以把线程当作是程序的子程序。
 
每一个应用程序都必须至少包含一个线程。在openFrameworks中，这个不可少的线程就是 setup/update/draw 。我们称之为主线程(或者openGL线程)。我们可以创建多个线程，而它们之间相互独立。
 
因此， 当我们想要在程序中期读取一张图片，不需要在update中读取图片，
我们可以创建一个专门用于读取图片的副线程。问题上，一旦创建线程，
主线程就不知道什么时候这个副线程会结束，因此，我们需要一个得让主副线程得以交流。
还有的问题上，多个线程不能访问同事同一块内存。我们需要以下机制来确保不同线程实现异步访问内存。
 
首先，我们来看在openFrameworks中如何创建一个线程。

## ofThread
 
每个应用程序都有至少包含一个线程，也就是主线程（也称之为openGL线程），用来调用openGL。
 
而我们对于需要耗费长时间的任务，则应该使用辅助线程。
在openFrameworks中，我们可以通过ofThread类来使用额外的线程。
ofThread并不能直接使用，需要我们继承它，并根据需求实现 `threadedFunction` ，最后，在需要的时候激活这个线程。

```cpp
class ImageLoader: public ofThread{ //继承ofThread
    void setup(string imagePath){ 
        this->path = imagePath;
    }

    void threadedFunction(){ //线程的主要工作
        ofLoadImage(image, path);
    }

    ofPixels image;
    string path;
}

//ofApp.h

ImageLoader imgLoader;


// ofApp.cpp
void ofApp::keyPressed(int key){
    imgLoader.setup("someimage.png");
    imgLoader.startThread(); //激活线程
}
```
 
一旦我们调用 `startThread()`，`ofThread`就会创建一个新的线程并立即返回。而这个线程则会立即调用 `threadedFunction`直到完成。
 
这样，就能在update进行的时候，同时进行读取图片，而不需中断。
  
现在，我们如何知道图片已读取完毕？这个线程可是独立与我们的主线程的。

![Simple Thread](images/simple_thread.png "Simple Thread")
 
如我们在上图所示，副线程并不会将在读取图片的情况主动告知主线程，而当其完成了读取图片，我们则可以通过确认其是否在运行来推断图片是否读取完毕，这需要用到方法 `isThreadRunning()`：

```cpp
class ImageLoader: public ofThread{
    void setup(string imagePath){
        this->path = imagePath;
    }

    void threadedFunction(){
        ofLoadImage(image, path);
    }

    ofPixels image;
    string path;
}

//ofApp.h
bool loading;
ImageLoader imgLoader;
ofImage img;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(loading==true && !imgLoader.isThreadRunning()){//线程是否在运行
        img.getPixelsRef() = imgLoader.image;
        img.update();
        loading = false;
    }
}

void ofApp::draw(){
    if (img.isAllocated()) {
        img.draw(0, 0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        loading = true;
        imgLoader.startThread();
    }
}
```

 
现在你知道如何读取一张图片了。
那如果我们一次性要读多张图片呢？一个可行的方法是创建多个线程。

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loading = false;
    }

    void load(string imagePath){
        this->path = imagePath;
        loading = true;
        startThread();
    }

    void threadedFunction(){
        ofLoadImage(image,path);
        loaded = true;
    }

    ofPixels image;
    string path;
    bool loading;
    bool loaded;
}

//ofApp.h
vector<unique_ptr<ImageLoader>> imgLoaders; //用vector创建线程组
vector<ofImage> imgs; //ofImage也要多个

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    for(int i=0;i<imgLoaders.size();i++){//逐个检查线程是否完成
        if(imgLoaders[i].loaded){
            if(imgs.size()<=i) imgs.resize(i+1);

            imgs[i].getPixelsRef() = imgLoaders[i].image;
            imgs[i].update();
            imgLoaders[i].loaded = false;
        }
    }
}

void ofApp::draw(){
    for(int i=0;i<imgLoaders.size();i++){
        imgs[i].draw(x,y);
    }
}

void ofApp::keyPressed(int key){
    imgLoaders.push_back(move(unique_ptr<ImageLoader>(new ImageLoader)));
    imgLoaders.back().load("someimage.png");
}
```

另一种办法只使用一个线程，但需要在线程中设置一个队列，才能让线程可以逐个读取图片。
我们需要在主线程中将图片路径添加到队列中，而辅助线程中的 `threadedFunction()` 则负责检查是否有新图片需要读取，如果有，则取出队列中的图片路径，并据此，将图片读取出来。

The problem with this is that we will be trying to access 
the queue from 2 different threads, and as we've mentioned 
in the memory chapter, when we add or remove elements to 
a memory structure there's the possibility that the memory
 will be moved somewhere else. If that happens while one 
 thread is trying to access it we can easily end up with 
 a dangling pointer that will cause the application to 
 crash. Imagine the next sequence of instruction calls 
 from the 2 different threads:

但这种方法存在一个问题：有两个不同的线程(主线程和辅助线程)都需要访问同一个内存区（这里是保存图片路径的队列），
如果一个线程需要修改该内存区的内容时（例如，添加新的图片路径），此时该内存区却被另一个线程修改（例如，移除旧的图片路径），那么，其中一个线程很可能读取到错误的信息，而这，会导致程序崩溃。
下一部分的教程演示了如何安全地从两个线程访问同一个内容。

        loader thread: finished loading an image //loader thread 结束读取图片
        loader thread: pos = get memory address of next element to load //pos=下一个需要读取的元素的地址
        main thread:   add new element in the queue //添加新的元素到队列
        main thread:   queue moves in memory to an area with enough space to allocate it //为了有足够的空间可以容纳新的元素，队列的内存地址发生改变
        loader thread: try to read element in pos  <- crash pos is no longer a valid memory address //尝试读取pos上的指向的元素 <- 崩溃， 现在pos不再是一个可访问的内存地址，


在这里，线程1`loader thread` 和线程2`main thread`会同时访问一个内存地址`pos`，但由于不能很好地安排其访问顺序，系统会终止它们并报出segmentation fault错误（原因是其中一个线程 如 线程1 对该地址的修改，例如，增加新的数据，导致分配的内存位置不足，因此，系统配了新的内存区，并注销了原来的内存区，但就在这个时候，另一个进程， 线程2 开始了对该内存区的访问，并且，由于未更新内存区的位置，导致线程2对老的内存区进行了访问，但由于老的内存区已经被注销了，于是，系统就立即报错）。


为了确保不同线程访问同一个内存可以井然有序，我们需要一个锁，当线程1操作的时候，给该线程上锁，则其他线程不能操作。
这种锁，即是c++中的mutex，也就是openFrameworks中的ofMutex。

在介绍mutex之前，我们还需要了解下thread与openGL。

## Threads and openGL

 
你可能会注意到前面：

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loaded = false;
    }
    void setup(string imagePath){
        this->path = imagePath;
    }

    void threadedFunction(){
        ofLoadImage(image, path);
        loaded = true;
    }

    ofPixels image;
    string path;
    bool loaded;
}

//ofApp.h
ImageLoader imgLoader;
ofImage img;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(imgLoader.loaded){
        img.getPixelsRef() = imgLoader.image;
        img.update();
        imgLoader.loaded = false;
    }
}

void ofApp::draw(){
    if (img.isAllocated()) {
        img.draw(0, 0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        imgLoader.startThread();
    }
}
```
 
在副线程中，不使用ofImage，而使用ofPixels来读取图片。而是在主线程再将ofPixels的内容放入ofIamge中。这是因为，古老的openGL，一次只能和一个线程工作。而这也是为什么我们直接称主线程为GL线程。


在之前的高级图形（advanced graphics）中和ofbooks的其他章节中。openGL是典型的异步工作，客户端／服务器模式。我们的应用程序时客户端，用来发送信息，告诉服务器openGL需要显示什么，然后，openGL则会根据其实际情况，将需要显示的图形发送给显卡。


也正是因为这个原因，openGL可以很好地和一个线程协作，也就是主进程。但如果尝试从不同线程调用openGL，则必然会导致程序崩溃，至少不会显示想要的图片信息。


而当调用ofImage的`img.loadImage(path)`,它实际上调用了openGL来处理图片的纹理。而如果我们在非GL线程调用它，那么，应用程序就会崩溃，或者，不会正确显示纹理。


当然，我们也可以告诉ofImage，和openFrameworks中其他需要使用像素和纹理的对象，不要使用像素，并且，不要调用openGL读取纹路，这样，就可以在非GL线程下使用ofImage了。如下：

```cpp
class ImageLoader: public ofThread{
    ImageLoader(){
        loaded = false;
    }
    void setup(string imagePath){
        image.setUseTexture(false);//禁用纹路
        this->path = imagePath;
    }

    void threadedFunction(){
        image.loadImage(path);
        loaded = true;
    }

    ofImage image;
    string path;
    bool loaded;
}

//ofApp.h
ImageLoader imgLoader;

// ofApp.cpp
void ofApp::setup(){
    loading = false;
}

void ofApp::update(){
    if(imgLoader.loaded){
        imgLoader.image.setUseTexture(true);
        imgLoader.image.update();
        imgLoader.loaded = false;
    }
}

void ofApp::draw(){
    if (imgLoader.image.isAllocated()){
        imgLoader.image.draw(0,0);
    }
}

void ofApp::keyPressed(int key){
    if(!loading){
        imgLoader.setup("someimage.png");
        imgLoader.startThread();
    }
}
```


In general remember that accessing openGL outside of the GL thread is not safe. 
In openFrameworks you should only do operations that involve openGL calls from the main thread, that is, from the calls that happen in the setup/update/draw loop, the key and mouse events, and the related ofEvents. 
If you start a thread and call a function or notify an ofEvent from it, that call will also happen in the auxiliary thread, so be careful to not do any GL calls from there.

在不同线程中，有许多方法可以安全使用openGL，例如，创建一个可分享区域，用来给不同的线程读取纹路。或者使用PBO来规划内存（不过这超出本章节讨论范围）。
总体而言，在openGL线程外访问openGL是很不安全的。在openFrameworks，你应该只在主线程的`setup()`/`update()`/`draw()`,`key event`和 `mouse event` 及其他标准的`ofeEvent`中调用openGL相关的操作。在其他线程中，如果调用了ofEvent，也需要确保其没有调用openGL。


另外，对于声音，也需要多加留意，ofSoundStream会创建其单独的线程来保证声音的准确输出，因此，在使用ofSoundStream，也需要小心。

## ofMutex

Before we started the openGL and threads section we were talking 
about how accessing the same memory area from 2 different threads 
can cause problems. 
This mostly occurs if we write from one of the threads causing
 the data structure to move in memory or make a location invalid. 



To avoid that we need something that allows to access that data to 
only one thread simultaneously. For that we use something called 
mutex. When one thread want's to access the shared data, it locks 
the mutex and when a mutex is locked any other thread trying to 
lock it will get blocked there until the mutex is unlocked again. You can think of this as some kind of token that each thread needs to have to be able to access the shared memory. 

Imagine you are with a group of people building a tower of cards,
 if more than one at the same time tries to put cards on it it's 
 very possible that it'll collapse so to avoid that, anyone who 
 wants to put a card on the tower, needs to have a small stone, 
 that stone gives them permission to add cards to the tower
  and there's only one, so if someone wants to add cards 
  they need to get the stone but if someone else has the 
  stone then they have to wait till the stone is freed. 
  If more than one wants to add cards and the stone is
  not free they queue, the first one in the queue gets
   the stone when it's finally freed.

A mutex is something like that, to *get the stone* you call lock on the mutex, 
once you are done, you call unlock. 
If some other thread calls lock while another 
thread is holding it, they are put in to a queue, 
the first thread that called lock will get the mutex when it's finally unlocked:

        thread 1: lock mutex
        thread 1: pos = access memory to get position to write
        thread 2: lock mutex <- now thread 2 will stop it's execution till thread 1 unlocks it so better be quick
        thread 1: write to pos
        thread 1: unlock mutex
        thread 2: read memory
        thread 2: unlock mutex

When we lock a mutex from one thread and another thread tries to lock it,
 that stops it's execution. For this reason we should try to do 
 only fast operations while we have the mutex locked in order to not lock 
 the execution of the main thread for too long.

In openFrameworks, the ofMutex class allows us to do this kind of locking. 
The syntax for the previous sequence would be something like:

        thread 1: mutex.lock();
        thread 1: vec.push_back(something);
        thread 2: mutex.lock(); // now thread 2 will stop it's execution until thread 1 unlocks it so better be quick
        thread 1: // end of push_back()
        thread 1: mutex.unlock();
        thread 2: somevariable = vec[i];
        thread 2: mutex.unlock();


We just need to call `lock()` and `unlock()` 
on our ofMutex from the different threads, 
from `threadedFunction` and from the update/draw loop
 when we want to access a piece of shared memory. 
 ofThread actually contains an ofMutex that can be
  locked using lock()/unlock(), we can use it like:

```cpp
class NumberGenerator{
public:
    void threadedFunction(){
        while (isThreadRunning()){
            lock();
            numbers.push_back(ofRandom(0,1000));
            unlock();
            ofSleepMillis(1000);
        }
    }

    vector<int> numbers;
}

// ofApp.h

NumberGenerator numberGenerator;

// ofApp.cpp

void ofApp::setup(){
    numberGenerator.startThread();
}

void ofApp::update(){
    numberGenerator.lock();
    while(!numberGenerator.numbers.empty()){
        cout << numberGenerator.numbers.front() << endl;
        numberGenerator.numbers.pop_front();
    }
    numberGenerator.unlock();
}
```

As we've said before, when we lock a mutex we stop other threads from accessing it. It is important that we try to keep the lock time as small as possible or
 else we'll end up stopping the main thread anyway making the 
 use of threads pointless.

## External threads and double buffering

Sometimes we don't have a thread that we've created ourselves, 
but instead we are using a library that creates 
it's own thread and calls our application on a callback.
 Let's see an example with an imaginary video library that
  calls some function whenever there's a new frame from the camera, 
  that kind of function is called a callback because some
   library *calls us back* when something happens, 
   the key and mouse events functions in OF are examples of callbacks.

```cpp
class VideoRenderer{
public:
    void setup(){
        pixels.allocate(640,480,3);
        texture.allocate(640,480,GL_RGB);
        videoLibrary::setCallback(this, &VideoRenderer::frameCB);
        videoLibrary::startCapture(640,480,"RGB");
    }

    void update(){
        if(newFrame){
            texture.loadData(pixels);
            newFrame = false;
        }
    }

    void draw(float x, float y){
        texture.draw(x,y);
    }

    void frameCB(unsigned char * frame, int w, int h){
        pixels.setFromPixels(frame,w,h,3);
        newFrame = true;
    }

    ofPixels pixels;
    bool newFrame;
    ofTexture texture;
}
```

Here, even if we don't use a mutex, our application won't crash.
 That is because the memory in pixels is preallocated in setup 
 and it's size never changes. For this reason the memory 
 won't move from it's original location. The problem is
  that both the update and frame_cb functions might be 
  running at the same time so we will probably end
   up seeing [tearing](http://en.wikipedia.org/wiki/Screen_tearing).
    Tearing is the same kind of effect we can see when we draw to
     the screen without activating the vertical sync.

To avoid tearing we might want to use a mutex:

```cpp
class VideoRenderer{
public:
    void setup(){
        pixels.allocate(640,480,3);
        texture.allocate(640,480,GL_RGB);
        videoLibrary::setCallback(this, &VideoRenderer::frameCB);
        videoLibrary::startCapture(640,480,"RGB");
    }

    void update(){
        mutex.lock();
        if(newFrame){
            texture.loadData(pixels);
            newFrame = false;
        }
        mutex.unlock();
    }

    void draw(float x, float y){
        texture.draw(x,y);
    }

    void frameCB(unsigned char * frame, int w, int h){
        mutex.lock();
        pixels.setFromPixels(frame,w,h,3);
        newFrame = true;
        mutex.unlock();
    }

    ofPixels pixels;
    bool newFrame;
    ofTexture texture;
    ofMutex mutex;
}
```


That will solve the tearing, but we are stopping the main thread while the `frameCB` is updating the pixels and stopping the camera thread while the main one is uploading the texture. For small images this is usually ok, but for bigger images we could loose some frames. A possible solution is to use a technique called double or even triple buffering:


```cpp
class VideoRenderer{
public:
    void setup(){
        pixelsBack.allocate(640,480,3);
        pixelsFront.allocate(640,480,3);
        texture.allocate(640,480,GL_RGB);
        videoLibrary::setCallback(this, &VideoRenderer::frameCB);
        videoLibrary::startCapture(640,480,"RGB");
    }

    void update(){
        bool wasNewFrame = false;
        mutex.lock();
        if(newFrame){
            swap(pixelsFront,pixelsBack);
            newFrame = false;
            wasNewFrame = true;
        }
        mutex.unlock();

        if(wasNewFrame) texture.loadData(pixelsFront);
    }

    void draw(float x, float y){
        texture.draw(x,y);
    }

    void frameCB(unsigned char * frame, int w, int h){
        pixelsBack.setFromPixels(frame,w,h,3);
        mutex.lock();
        newFrame = true;
        mutex.unlock();
    }

    ofPixels pixelsFront, pixelsBack;
    bool newFrame;
    ofTexture texture;
    ofMutex mutex;
}
```

With this we are locking the mutex for a very short time in the frame callback to set `newFrame = true` in the main thread. We do this to check if there's a new frame and then to swap the front and back buffers. `swap` is a c++ standard library function that swaps 2 memory areas so if we swap 2 ints `a` and `b`, `a` will end up having the value of `b` and viceversa, usually this happens by copying the variables but `swap` is overridden for ofPixels and swaps the internal pointers to memory inside `frontPixels` and `backPixels` to point to one another. After calling `swap`, `frontPixels` will be pointing to what `backPixels` was pointing to before, and viceversa. This operation only involves copying the values of a couple of memory addresses plus the size and number of channels. For this reason it's way faster than copying the whole image or uploading to a texture.

Triple buffering is a similar technique that involves using 3 buffers instead of 2 and is useful in some cases. We won't see it in this chapter.

## ofScopedLock

Sometimes we need to lock a function until it returns, or lock 
for the duration of a full block. That is exactly what a 
scoped lock does. If you've read the memory chapter 
you probably remember about what we called 
initially, *stack semantics*, or 
RAII [Resource Adquisition Is Initialization](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization). 
A scoped lock makes use of that technique to lock a mutex for the whole duration of the block, even any copy that might happen in the same `return` call if there's one.

For example, the previous example could be turned into:

```cpp
class VideoRenderer{
public:
    void setup(){
        pixelsBack.allocate(640,480,3);
        pixelsFront.allocate(640,480,3);
        texture.allocate(640,480,GL_RGB);
        videoLibrary::setCallback(&frame_cb);
        videoLibrary::startCapture(640,480,"RGB");
    }

    void update(){
        bool wasNewFrame = false;

        {
        ofScopedLock lock(mutex);
            if(newFrame){
                swap(fontPixels,backPixels);
                newFrame = false;
                wasNewFrame = true;
            }
        }

        if(wasNewFrame) texture.loadData(pixels);
    }

    void draw(float x, float y){
        texture.draw(x,y);
    }

    static void frame_cb(unsigned char * frame, int w, int h){
        pixelsBack.setFromPixels(frame,w,h,3);
        ofScopedLock lock(mutex);
        newFrame = true;
    }

    ofPixels pixels;
    bool newFrame;
    ofTexture texture;
    ofMutex mutex;
}
```

A ScopedLock is a good way of avoiding problems because we forgot to unlock a mutex and allows us to use the `{}` to define the duration of the lock which is more natural to C++.

There's one particular case when the only way to properly lock is by using a scoped lock. That's when we want to return a value and keep the function locked until after the value was returned. In that case we can't use a normal lock:

```cpp

ofPixels accessSomeSharedData(){
    ofScopedLock lock(mutex);
    return modifiedPixels(pixels);
}

```

We could make a copy internally and return that later, but with this pattern we avoid a copy and the syntax is shorter.


## Poco::Condition

A condition, in threads terminology, is an object that allows to 
synchronize 2 threads. The pattern is 
something like this: one thread waits for something 
to happen before starting it's processing. 
When it finishes, instead of finishing the thread, 
it locks in the condition and waits till there's new data to process. 
An example of this could be the image loader class we were working 
with earlier. Instead of starting one thread for every image, we 
might have a queue of images to load. The main thread adds image
 paths to that queue and the auxiliary thread loads the images 
 from that queue until it is empty. The auxiliary thread then
  locks on a condition until there's more images to load.

Such an example would be too long to write in this section, 
but if you are interested in how something like that might 
work, take a look at ofxThreadedImageLoaded (which does just that).

Instead let's see a simple example. 
Imagine a class where we can push urls to
 pings addresses in a different thread. Something like:

```cpp
class ThreadedHTTPPing: public ofThread{
public:
    void pingServer(string url){
        mutex.lock();
        queueUrls.push(url);
        mutex.unlock();
    }

    void threadedFunction(){
        while(isThreadRunning()){
            mutex.lock();
            string url;
            if(queueUrls.empty()){
                url = queueUrls.front();
                queueUrls.pop();
            }
            mutex.unlock();
            if(url != ""){
                ofHttpUrlLoad(url);
            }
        }
    }

private:
    queue<string> queueUrls;
}
```

The problem with that example is that the auxiliary thread 
keeps running as fast as possible in a loop, 
consuming a whole CPU core from our computer which is not a very good idea.

A typical solution to this problem 
is to sleep for a while at the end of each cycle like:

```cpp
class ThreadedHTTPPing: public ofThread{
public:
    void pingServer(string url){
        mutex.lock();
        queueUrls.push(url);
        mutex.unlock();
    }

    void threadedFunction(){
        while(isThreadRunning()){
            mutex.lock();
            string url;
            if(queueUrls.empty()){
                url = queueUrls.front();
                queueUrls.pop();
            }
            mutex.unlock();
            if(url != ""){
                ofHttpUrlLoad(url);
            }
            ofSleepMillis(100);
        }
    }

private:
    queue<string> queueUrls;
};
```

That alleviates the problem slightly but not completely. 
The thread won't consume as much CPU now, 
but it sleeps for an unnecesarily while 
when there's still urls to load. It also 
continues to run in the background even when
 there's no more urls to ping. Specially 
 in small devices powered by batteries, 
 like a phone, this pattern would drain the battery in a few hours.

The best solution to this problem is to use a condition:

```cpp
class ThreadedHTTPPing: public ofThread{
    void pingServer(string url){
        mutex.lock();
        queueUrls.push(url);
        condition.signal();
        mutex.unlock();
    }

    void threadedFunction(){
        while(isThreadRunning()){
            mutex.lock();
            if (queueUrls.empty()){
                condition.wait(mutex);
            }
            string url = queueUrls.front();
            queueUrls.pop();
            mutex.unlock();

            ofHttpUrlLoad(url);
        }
    }

private:
    Poco::Condition condition;
    queue<string> queueUrls;
};
```

Before we call `condition.wait(mutex)` the mutex needs to be locked, then the condition unlocks the mutex and
 blocks the execution of that thread until `condition.signal()` is called. 
 When the condition awakens the thread because it's been signaled, 
 it locks the mutex again and continues the execution. 
 We can read the queue without problem because we know
  that the other thread won't be able to access it. 
  We copy the next url to ping and unlock the mutex to 
  keep the lock time to a minimum. Then outside the lock we 
  ping the server and start the process again.

Whenever the queue gets emptied the condition will 
block the execution of the thread to avoid it from running in the background.

## Conclusion

As we've seen threads are a powerfull tool to allow for several tasks to happen simultaneously 
in the same application. They are also hard to use, the main problem is usually accessing shared resouces, usually shared memory. 
We've only seen one specific case, how to use threads to do background tasks that will pause the execution of the main task,
 there's other cases where we can parallelize 1 task by dividing it in small subtasks like for 
 example doing some image operation by dividing the image in for subregions 
 and assigning a thread to each. For those cases there's special libraries 
 that make the syntax easier, OpenCv for example can do some operations
  using more than one core through [TBB](https://www.threadingbuildingblocks.org/) and
   there's libraries like the same TBB or [OpenMP](http://openmp.org/wp/) that allow 
   to specify that a loop should be divided and run simultaneol¡usly in more than one core
