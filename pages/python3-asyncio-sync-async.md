# Python3 asyncio - call async code from synchronous code

Python 3.4 introduced `asyncio` module and a new programming padigrim in the lagnuage - coroutines. However these coroutines are not exactly like one would expect. They are based on python generators. Code looks something like this:

```py
async def my_other_function():
    # < ... >
    return None

async def my_function():
    result = await my_other_function()
```

Since these are python generators we must preserve `async`/`await` chain through entire call stack. In simpler words: you can not `await` in the function that was defined *without* `async`. On the first sight this looks simply as inconvenience. Consider windows fibers or any other stackful coroutine implementation - it is possible to yield and give execution back to your event loop from arbitrary call stack location without need of any special keywords rippling through entire call stack.

# The problem

This is more than a simple inconvenience. Problem stems from the fact that while we can call `def` functions from `async def` functions, we can not do opposite. This essentially divides python code into two islands - async is land and synchronous one. Problems start to arise when we need to call async code from synchronous code which we do not control. For example consider synchronous library that invokes a user-defined callback. Invokation does not contain `await` therefore callback can not be defined as `async def`. Now we need to do some networking in this callback and get the result immediately. Unfortunately networking library we use is async, therefore networking calls must be awaited, which we can not do. Daniel Arbuckle raised [this issue](http://bugs.python.org/issue22239) about three years ago, but it was considered by BDFL as insignificant. Indeed nested event loops would help in this case as we could continue scheduling event loop in our synchronous code until coroutine returns a result. Python can not do this yet.

# The solution

Daniel Arbuckle was kind enough to come up with a proof of concept solution which is very simple as well. Some things have changed however - asyncio implements several things in c and in order to monkey-patch reentrant loops into python we must opt out from faster modules and use pure-python implementations. Below is python module that implements reentrant event loops. See end of the module for example usage.

```py
"""
Adapted to python 3.6 by Rokas Kupstys.
Based on code from http://bugs.python.org/issue22239 by Daniel Arbuckle.
License: PYTHON SOFTWARE FOUNDATION LICENSE VERSION 2 (i am sure Daniel Arbuckle will agree).
"""
import sys
import asyncio
import asyncio.tasks
import asyncio.futures


def run_nested_until_complete(future, loop=None):
    """Run an event loop from within an executing task.

    This method will execute a nested event loop, and will not
    return until the passed future has completed execution. The
    nested loop shares the data structures of the main event loop,
    so tasks and events scheduled on the main loop will still
    execute while the nested loop is running.

    Semantically, this method is very similar to `yield from
    asyncio.wait_for(future)`, and where possible, that is the
    preferred way to block until a future is complete. The
    difference is that this method can be called from a
    non-coroutine function, even if that function was itself
    invoked from within a coroutine.
    """
    if loop is None:
        loop = asyncio.get_event_loop()

    loop._check_closed()
    if not loop.is_running():
        raise RuntimeError('Event loop is not running.')
    new_task = not isinstance(future, asyncio.futures.Future)
    task = asyncio.tasks.ensure_future(future, loop=loop)
    if new_task:
        # An exception is raised if the future didn't complete, so there
        # is no need to log the "destroy pending task" message
        task._log_destroy_pending = False
    while not task.done():
        try:
            loop._run_once()
        except:
            if new_task and future.done() and not future.cancelled():
                # The coroutine raised a BaseException. Consume the exception
                # to not log a warning, the caller doesn't have access to the
                # local task.
                future.exception()
            raise
    return task.result()


def __reentrant_step(self, exc=None):
    containing_task = self.__class__._current_tasks.get(self._loop, None)
    try:
        __task_step(self, exc)
    finally:
        if containing_task:
            self.__class__._current_tasks[self._loop] = containing_task


def monkeypatch():
    global __task_step
    # Replace native Task, Future and _asyncio module implementations with pure-python ones. This is required in order
    # to access internal data structures of these classes.
    sys.modules['_asyncio'] = sys.modules['asyncio']
    asyncio.Task = asyncio.tasks._CTask = asyncio.tasks.Task = asyncio.tasks._PyTask
    asyncio.Future = asyncio.futures._CFuture = asyncio.futures.Future = asyncio.futures._PyFuture

    # Replace Task._step with reentrant version.
    __task_step = asyncio.tasks.Task._step
    asyncio.tasks.Task._step = __reentrant_step


if __name__ == '__main__':
    async def asynchronous_code():
        return 42

    def synchronous_code():
        return run_nested_until_complete(asynchronous_code())

    async def coroutine():
        print(synchronous_code())

    monkeypatch()
    loop = asyncio.get_event_loop()
    loop.run_until_complete(coroutine())
    loop.close()
```

