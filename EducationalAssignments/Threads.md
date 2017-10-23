# Threads and synchronization

This assignment introduces *concurrency*, a fundamental concept in
computer science. In Repy programs, concurrent actions are implemented
using *threads*. We will also introduce the *closure* technique for
passing call arguments to threads. Furthermore, there are primitives
to coordinate threads when they try to access a shared resource, i.e.
means of *synchronization*. Repy uses *locks* for this.

We assume you have a running copy of RepyV2 on your machine already.
If not, check out the [Build Instructions](../Contributing/BuildInstructions.md)
to create one.
Also, please keep the [RepyV2 API documentation](../Programming/RepyV2API.md)
at hand.


# Threads in Repy

In Repy, threads are Repy functions that run concurrently with the
other parts of a Repy program. The `createthread(functionname)`
API call causes a new thread for the function `functionname` to
be created. Repy calls this function with no call arguments,
`functionname()`, and the `createthread` call returns immediately
without waiting for the called function to return.

Indeed, a function running inside of a thread cannot use the
`return` statement to hand back values to its caller --- the
program flow on the caller's side just continues! Threads can
access functions with side effects (such as `log`) though, and
we will later discuss methods for programmatically getting data
out of threaded functions.

Another thing to keep in mind is that there are no guarantees
as to the exact start time of a thread, and how much computational
time and other resources it will be assigned before another
thread is scheduled. However, unless the thread voluntarily finishes
or encounters an exception, it will get its turn again and again
over the course of program execution.

Here is a synthetic example that demonstrates Repy's `createthread`
call and the peculiarities described.

```python
# We will launch this function as a separate thread soon
def print1():
  one = "1\n"
  while True:
    log(one)

# Create the new thread; it probably takes a while to actually start
createthread(log1)
# The `createthread` call immediately returns. (If we just called
# the `print1` function, the control flow would *never* return.)

# For comparison, do a similar thing in the main context too
two = "2\n"
while True:
  log(two)
```

If you run this little program on your machine a few times, you will
likely see emanations of all of the effects described earlier: Sometimes
the "ones" thread starts earlier, sometimes the other; sometimes the
two numbers alternate rapidly in the output, sometimes there are longer
runs of a number. By and large, both numbers will probably show up
approximately equally frequently.



# Passing Data To Threads Using Closures

We mentioned that the `createthread` function in Repy takes another
function as its argument and calls it, but it does not pass arguments
to this function. You can easily verify this with a short program.

```python
# "*args" is Python's (and Repy's) way to mark a function to take
# variadic arguments.
# The function prints whatever arguments that caller sends.
def whataremyargs(*args):
  log(args)

# Create a new thread with the above function
createthread(whataremyargs)
```

Running this program you see that the function in the thread is given
an empty tuple `()` for its arguments. So, if we cannot pass arguments
to the function in the usual way, can we at all?

Indeed there are two ways to do this, and both depend on Python's
scoping rules to work. You have probably seen already that variables
from an enclosing scope are readble from an inner scope too. For example:

```python
# This is the outer scope
myvar = 7

def justread():
  # This is the inner scope of the function. The variable from the
  # outer scope is visible just fine
  log(myvar)

justread()

# If the variable is modified, the function sees the updated value.
myvar = -1000

justread()
```

A function that you launch as a thread can thus read variables defined
in the calling scope to pass parameters. Very often, it is useful to
define functions for threads from the same template, but pass (via the
outer scope) different data into them. The pattern to use for this is
called *closure*: We declare a function (which takes arguments that we
want to pass) to return a function (which takes no arguments and instead
accesses its outer scope).

```python
# This function returns an argumentless function that we can pass on
# to `createthread`.
def create_sleep_function(seconds_to_sleep):
  # Define the inner function; note that we use the call argument
  # from the enclosing scope
  def customized_sleep_function():
    sleep(seconds_to_sleep)
  # The function is defined, return it
  return customized_sleep_function

# Create a few of these functions and test them 
for i in range(2,6):
  log("Going to sleep for", i, "seconds at", getruntime(), "...")
  a_function = create_sleep_function(i)
  a_function()
  log("and waking up at", getruntime(), "again\n")
```

In contrast to the previous example, this construct fixes the number
of seconds of sleep for every returned function at the time we call
the `create_sleep_function`. This is because the parameter to the
creating function ceases to exist when the function returns ---
we have no outer-scope access to it anymore.



What if we want to modify
the outer-scope variables? This program will fail because when Repy
(and Python) detects a modification to a variable, it will automatically
assume that the variable must be local!

```python
myvar = 25

def trytomodify():
  # You would think that this works, but the code modifying the variable
  # makes the variable local-scoped, and there is no definition for the
  # variable yet!
  log(myvar)
  myvar += 1
  log(myvar)

trytomodify()
```

This program errors out with an `UnboundLocalError`.




# Locks in Repy


