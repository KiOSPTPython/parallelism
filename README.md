# Multiprocessing and Async
## Multiprocessing
With the `multiprocessing` library we can achieve parallelism in our code. It gives us a few classes to work with when structuring our code. First, the `Process` class. The `Process` class surrounds a piece of code and runs in a separate process when started. When we start a `Process` object, it will execute whatever code we gave it on instantiation. There are two ways we can provide it with instructions on what to do. Either we pass a function argument to the class or subclass `Process` with our class and override the method `run()`. Let's see this in action.
```python
from multiprocessing import Process


def gump():
    """Time to take off."""
    for _ in range(5):
        print("Running and")


if __name__ == '__main__':
    p1 = Process(target=gump)
    p2 = Process(target=gump)
    p1.start()
    p2.start()
    p1.join()
    p2.join()
```
We see two things. First, how to define a `Process`. We instantiated the object with our target function as its only keyword argument. This creates a `Process` object that, when started, will execute the function `gump()`. Second, how to start the new process. We called `start()` on each object to instruct it to begin executing. And we called `join()` to wait for the process to finish. It is standard practice to call `join()` on each started `Process` or else we may leave them in a zombie state when our program exits. Finally, note that the `Process` object inherits some of the parent process resources, such as the stdout to which it can write (prints to our screen).


```python
from multiprocessing import Process


def gump(msg):
    """Time to take off."""
    for _ in range(5):
        print(msg)


if __name__ == '__main__':
    p1 = Process(target=gump, args=('Shrimping', ))
    p1.start()
    p1.join()

```
Another example of the same thing in which we also pass an argument to the function the subprocess will execute. So long as we send at least one arugment, `args` must be a tuple.


```python
from multiprocessing import Process


class Spider(Process):
    def __init__(self, address):
        super().__init__()
        self.addr = address

    def run(self):
        """Fake spider algorithm."""
        print(f"Crawling through {self.addr}.")


if __name__ == '__main__':
    s1 = Spider("www.fanbase.com")
    s2 = Spider("www.com")
    s1.start()
    s2.start()
    s1.join()
    s2.join()
```
This code subclasses `Process` as another method to define work for it. Whatever code we want executed must appear in the `run()` method. Not to be confused by the `start()` method which we call to run the subprocess. When we subclass `Process` and use its `__init__()` method, we must call its parent `__init__()` before anything else or else things break. When we instantiated our `Spider`, in the above example, we also gave it an address to work with; Currently, just printing it to screen. 

>Write a function that receives a task_name argument and prints its value to the screen. Then, start 3 `Process` objects that were instatiated using the first form. Notice that if you print in a loop in each process, the different outputs become erratic.


Each subprocess created this way does not share data with the parent process or other subprocesses. We can create autonomous tasks and run them in parallel which can help us in CPU intensive programs but we, sometimes, want to share data or communicate. 
## Communication breakdown
Because the need to communicate between subprocesses is a real one, the library `multiprocessing` provides us with a couple of ways to do it. We'll play around with both. Each have their own advantages and suit different designs. The first way to go about it is using `multiprocessing.Queue`. A queue is a data structure that allows items to be added and removed from it. Items are added to the beginning of a queue and removed from the end of it. In other words, a Queue object works just like a queue at a store. Those who were first in, are the first out (FIFO). Examples speak louder than words; Let's look at one.
```python
from time import sleep
from multiprocessing import Process, Queue


def sleepy(q, doze_off):
    """A sleepy worker waits a little before waking up."""
    sleep(doze_off)
    q.put("Worker awake.")


def work(q):
    """Takes a message from the given Queue and prints it."""
    msg = q.get()
    print(f"Got '{msg}' from Queue.")


def main():
    q = Queue()
    worker1 = Process(target=sleepy, args=(q, 3))
    worker2 = Process(target=work, args=(q, ))
    worker1.start()
    worker2.start()
    worker1.join()
    worker2.join()


if __name__ == '__main__':
    main()
```
The `Queue` object is shared between all processes and can be used by both subprocesses and parent process. What we did here is create a `Queue`, pass it to the `Process` constructors and allow both subprocesses to communicate with each other by passing messages. Both `put()` and `get()` operations can be used in parallel safely. Passing around a mutable object will not yield the same results, such as `dict`. When we call `get()` on an empty `Queue`, the code will block until an item is available. Similarly, `put()` will block if the `Queue` is full (has no limit, by default).


Alternatively, we can create connection objects using `Pipe` that enable communication. These, too, are sharable objects that, when passed to the subprocess[es], can be used to send and receive data. Calling `Pipe` creates two communication objects, or two "ends" of the same pipe, to which a subprocess (or parent process) can read/write with the methods `recv()` and `send()`, respectively. Whatever is written on one end of the pipe will be available on the other end. Calling `recv()` blocks the code until data is available.
```python
from multiprocessing import Process, Pipe


def alice(pipe_right):
    """Alice can talk to someone on the other side."""
    bob_said = pipe_right.recv()
    print(f"I heard Bob say '{bob_said}.'")
    pipe_right.send("Roger roger. Over and out.")


def bob(pipe_left):
    """Bob can talk to someone on the other side."""
    pipe_left.send("Hello, Alice!")
    alice_said = pipe_left.recv()
    print(f"I heard Alice say '{alice_said}'.")


def main():
    """Create a pair of communication obj for Bob and Alice."""
    left, right = Pipe()
    pbob = Process(target=bob, args=(left, ))
    palice = Process(target=alice, args=(right, ))
    pbob.start()
    palice.start()
    pbob.join()
    palice.join()
    left.close()
    right.close()


if __name__ == '__main__':
    main()
```
Both `bob()` and `alice()` use a connection object to send and receive data. The first thing we did is create these connections using `Pipe()` which returns a tuple of the two objects. One we nick named left and the other right. We sent each to our Bob and Alice to permit them to talk to one another. After we are done with the connections, we should close them.

>Write a program with 2 workers and one "manager". The manager pushes messages into a Queue, shared by all 3 subprocesses. Each worker prints a message from the queue and sleeps for 0.1 seconds. The manager communicates, in an infinite loop, with the parent process, using a connection object returned by `Pipe()`, to know what work to give the workers. The parent process asks the user for "work" the workers should print. 


If you're brave, devise a way to gracefully shutdown all workers and manager. Hint: The shutdown should come from a user's input.

## Locks
You cannot talk about parallelism without talking about locks. Sometimes, in multiprocessing (and threading) we want to synchronize between subprocesses and not let them run wild with their tasks. Imagine you want to avoid one task from being executed in parallel, such as printing to the screen. Instead, you want to print to the screen in an orderly fashion. Let's look at an example straight out of the docs (almost).
```python
from multiprocessing import Process, Lock


def f(l, i):
    """Acquire the lock, l, and print 'Hello, world!'"""
    with l:
        print("Hello, world ", i)


if __name__ == '__main__':
    lock = Lock()
    for num in range(10):
        Process(target=f, args=(lock, num)).start()
```
The way this works is, firstly, `Lock` is shared. Its state, at any given time, is known to all who try to use it. When the the lock is acquired, or "closed", no one else can acquire the lock and must wait when they try to do so. They wait until the lock is released, or "unlocked". As soon as the lock is released, whoever is waiting to acquire it, will acquire it and code execution will continue. In other words, try to acquire the lock is a blocking operation. We did this in the `with` invocation. 

>Refactor your previous code, with the workers and manager, to allow only a single process to ever write to screen in parallel.

## Asynchronous and asyncio
Asynchronous programming's primary focus is to optimize network programs. If we have 3, independent, functions that run sequentially - A, B and C. But, function B has to wait for data to be available from a socket, function C will have to wait for B to finish. This creates a time span in which our CPU is idle while it could have finished C's execution (while waiting for B to be ready). What asynchronous programming gives us is the ability to put aside a function execution for later. Either when an I/O resource is ready, another async function is ready or just a date event. The basic tools to implement this are in Python's `asyncio` library.


There are a few core concepts we need to understand that will help us use asynchronous programming in Python. Namely, event loop, coroutine, futures and tasks.

## Event loop
The event loop is our reactor. It reacts to events that occur during execution. Imagine a real world example like cooking a simple omelette with vegetables. While the eggs are on the stove, you cut the vegetables. When the eggs are half done, you add the vegetables. The event that we reacted on is our eggs being half done and at the right time to add the vegetables. In software, we don't have a lot of vegetables but, we can react to the kernel's events of I/O. For example, when a socket has data and is ready to read, the kernel sends an event about it. The object that is resposible for managing what to do on what event is called the event loop. It is meant to run as long as our program is up. The event loop is accessibly anywhere in the code like so.
```python
import asyncio


loop = asyncio.get_event_loop()
```
Once we have this object, we can use it to structure our program. Starting with chaining events together such as, on new socket opened do a task. Tasks' first building block is a coroutine.

## Coroutines
A coroutine is a lightweight suspended function. It is something that is meant to execute in the future under the event loop - We don't call it directly. We can wait for it to finish as soon as possible or chain it with other coroutines. Let's see this in action.
```python
import asyncio


async def wait_here(s):
    await asyncio.sleep(s)
    return "Done."


async def show_me():
    result = await wait_here(2)
    print(result)


loop = asyncio.get_event_loop()
loop.run_until_complete(show_me())
loop.close()
```
Writing coroutines is done with `async def` and they can either `return` a value for another coroutine waiting on their result, raise an exception or `await` another coroutine's completion. What we did here was tell the event loop to run a coroutine `show_me()`. The coroutine `show_me()` awaited the completion of coroutine `wait_here(2)` which, in turn, awaited the completion of `asyncio.sleep(5)`. We chained all 3 together. We cannot use `time.sleep(5)` because that is a blocking function. The coroutine `show_me()` used the result returned from `wait_here(s)` and displayed it to screen.
>Write your own async code that chains two coroutines together. One that receives a URL and extracts its domain, the other that prints the result.

## Futures and tasks
Another lego piece is `Future`. It is an object that promises to tell us what happened with some piece of work we assigned to it. It can tell us if it, the `Future`, is marked as done, store a result or even define what to do when the future object is marked as done, e.g. call a function on the event the `Future` is done. When we use a `Future` to wrap a coroutine, its work, we create the final piece of the puzzle, `Task`. Because `Task` is a subclass of `Future`, it has all of its methods, and then some. Let's look at an example of how to use these.
```python
import asyncio


async def wait_here(s):
    """A fake slow job that takes 5 seconds and returns 'Done.'"""
    await asyncio.sleep(s)
    return "Done."


async def show_me(f):
    """Receives a Future and marks it as done before the coroutine returns."""
    result = await wait_here(1)
    f.set_result(result)


def finished(future):
    """Prints a Future's result."""
    print(future.result())


loop = asyncio.get_event_loop()
future = asyncio.Future()
task = loop.create_task(show_me(future))
future.add_done_callback(finished)

loop.run_until_complete(task)
loop.close()
```
We did a few things here. First, we created a `Future` object that'll help tell us when the coroutine finished. When we called its method `set_result()`, inside the coroutine `show_me()`, we marked it as done implicitly. We scheduled a `Task` to run under our event loop. The task is made out of the coroutine `show_me()` and uses the `Future` we just created to let us know when it finishes. Lastly, before starting the event loop, we asked that a function call be made to `finished()` when our `Future` is done. The function `finished()` prints whatever is stored inside our `Future`. Previously, when we used `run_until_complete()` with a coroutine, it was made into a `Task` object for us.
>Refactor the above code and add a callback to task itself without using the `Future` object.


## Echo bot
Let's look at a classic echo server example. We'll create a server that echoes back whatever is sent to it.
```python
import asyncio


async def handle_echo(reader, writer):
    """
    Receives a read and write stream to a newly received socket.
    Echoes back the first 100 bytes read.
    """
    data = await reader.read(100)
    message = data.decode()
    addr = writer.get_extra_info('peername')
    print(f"Received {message} from {addr}")

    print(f"Send: {message}")
    writer.write(data)
    await writer.drain()

    print("Close the client socket")
    writer.close()


loop = asyncio.get_event_loop()
coro = asyncio.start_server(handle_echo, '127.0.0.1', 8888, loop=loop)
server = loop.run_until_complete(coro)

# Serve requests until Ctrl+C is pressed
sock_name = server.sockets[0].getsockname()
print(f"Serving on {sock_name}.")
try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

# Close the server
server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```

## Encoding
This topic can be a real headache, if you've encountered it before, and there's no going around it - We have to cover it to some extend to code correctly. 


All data that rests on disk or is coming through an open socket is made up of bytes. No surprise there. But, if we print the content of a file to screen, we don't see bytes. We see human readable symbols. This is because someone told us how the data is encoded on the file. For example `print(open('/etc/passwd', encoding='ascii').read())` prints the content of `/etc/passwd` and it can do so because it was told how to treat the bytes in the file. We told it to treat the bytes in the file as if they were encoded using an encoding called `ascii`. Ascii is a veteran encoding containing, among others, the alphanumeric range of characters. If we don't explicitly tell `open()` what encoding to use, it'll use the system default. This is also the reason why reading files with `open()` worked so far - It took the system default and it, by coincidence, was good enough.


The same applies to any source of data, including network. If we play around with sockets and incoming data, it will not be in the form of a string but in the form of bytes. We decode those bytes into something usable by telling Python the bytes are encoded in ascii (or whatever else).


The operation of translating bytes into readable symbols is decoding. We decode an array of bytes using the encoding 'utf-8', for example, into a readable format.


The operation of translating symbols into bytes is encoding. We encode into bytes using an encoding of our choosing, for example 'latin-1'.


Each encoding has its own rules. For example, `ascii` is limited to 256 symbols and each byte represents a symbol. But, `utf-8` has a much larger range. It has the same basic `ascii` mapping (same single bytes map to same symbol) in addition to more rules that enable it to map 2 bytes or even 4 bytes to different symbols.


The problem is, how do we manage so many encodings in our code? Does each object have to be attached to a property telling us what encoding it was/is? Not really, we use unicode. The object we send print is a unicode. In Python3 strings are unicode. Once we decoded bytes into this "usable" object, it becomes a unicode object.


To understand the relation betwteen bytes, encoding and unicode, consider the following. Encodings are dictionaries between bytes and symbols ({bytes: symbols}). Different encodings can have different byte "keys" to the same symbol. Unicode is a dictionary between integers and symbols. The unicode has room for 1.1 million such integers, called "code points". This has room for all symbols any encoding represents, so far. It covers all encodings out there. Only 110k code points have been taken up so far. 


The correct way to work is the Unicode sandwich, as nedbat calls it in his [excellent talk](https://nedbatchelder.com/text/unipain.html). As soon as data comes in, be it bytes from disk or bytes from network, we decode them into unicode. All work we have to do within our code is done using unicodes (remember strings in Python3 are unicode, such as `s = "hello"`). The moment we're ready to send data back outside into the world, disk/net or whatever else, we encode it using an encoding of our choosing.
Bytes on the outside, unicode on the inside.


Nedbat's talk is highly on the subject - [unipain](https://nedbatchelder.com/text/unipain.html).

## Ping bot
Let's change our code a little. This time, our server listens for connections on the same port but expects the data it receives to be an address. Then, it pings said address using [aioping](https://github.com/stellarbit/aioping).
```python
import asyncio
import aioping


async def boom(host):
    """Pings a given host address."""
    try:
        delay = await aioping.ping(host) * 1000
        # Format the value as a float and a single digit after decimal point.
        return f"Ping response in {delay:.1f} ms."
    except TimeoutError as err:
        return "Ping timed out."
    except IOError as err:
        return f"Pinging {host} failed. {err}"


async def handle_ping(reader, writer):
    """
    Receives a read and write stream to a newly received socket.
    Pings whatever address is given through the reader
    and sends back the response to the writer.
    """
    addr = writer.get_extra_info('peername')
    print(f"Got connection from {addr}.")

    data = await reader.read(100)
    host = data.decode().strip()

    print(f"Pinging {host}...")
    pinged = await boom(host)
    print(pinged)
    writer.write((pinged+'\n').encode('ascii'))
    await writer.drain()

    print(f"Closing client socket {addr}.")
    writer.close()


loop = asyncio.get_event_loop()
coro = asyncio.start_server(handle_ping, '127.0.0.1', 8888, loop=loop)
server = loop.run_until_complete(coro)

# Serve requests until Ctrl+C is pressed
sock_name = server.sockets[0].getsockname()
print(f"Serving on {sock_name}.")
try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

# Close the server
server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```
For each new client connecting to port 8888, we'll listen to any message it sends us, at a limit of 100 bytes. The server treats this message as an address it is meant to ping. It creates a new `Task` using the coroutine `boom(host)` and awaits its completion. Upon completion, the coroutine `boom()` would have returned the result of the ping. The coroutine `handle_ping()` will write the result using `writer` and finally close it before returning.

## Finals
Write an async server that waits for new clients to connect to it and give it instructions what to do using what value. It should listen on a port of your choosing and support two operations. 


The first is pinging, as we've done above on a given address. The second is HTTP GET on a given URL. You can use `aiohttp` to help you or `asyncio.open_connection`, if you're feeling brave. Remember that dictionary values can be functions. If you need to see how a HTTP GET request looks, use `nc -l -p 8080` and point your browser to yourself.