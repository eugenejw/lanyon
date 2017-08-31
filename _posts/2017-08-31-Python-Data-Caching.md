---
layout: post
title: The two flashlight paradox and taking advantage of Python's GC when doing data stream caching
---

> When switched from the language with manual memory management, such as C or C++, to a garbage-collected language, your job as a programmer is made much easier by the fact that your objects are automatically reclaimed when you’re through with them. It seems almost like magic when you first experience it, it can easily lead to the impression that you don’t have to think about memory management, but this isn’t quite true.

-- quote from *Effective Java* by **Joshua Bloch**.

Ignoring memory management in Python will probably not that often to lead you to situations where thing went very wrong -- However it still could. For example, passing tons of observer (recall) methods to observable objects, which do not explicitly destroy it(by nulling it out), when it became no longer needed. Overtime, the memory leak can happen.

Keeping memory management in mind will help you gain edges when dealing with "big data" tasks.

In this post, I am going to show you how to take the advantage of Garbage-Collection (GC) in Python.

## Stream Data Caching Problem
Sometimes, when data stream comes, we want to store them and preserve their natural order, for example, we keep them in order of the timestamp the data arrives.

Putting them as data nodes in a LinkedList is the common and natual implementation. 

But once we put them into the list, everytime when we need to access or retrieve one of the stored data nodes, it took O(logn) time to get to it.

Usually people build auxiliary HashMap to record the reference to all data nodes in the LinkedList. Thus we can gain the O(1) time complexity to search and access any node in the LinkedList.

This benefit comes with costs. When the hashmap grows, the collision rate increases, and the hashmap's performance will eventually deganerate to LinkedList or BST(if in Java8) as shown below, a quote from Ralf Sternberg's [blog](https://eclipsesource.com/blogs/2012/09/04/the-3-things-you-should-know-about-hashcode/), the hashCode collision reaches 50% when there are 100,000 distinct objects in the hashmap.
![img](https://eclipsesource.com/wp-content/uploads/2012/09/hashcode-collisions.png)

So we sometimes have to reduce the size of the cache in memory by, say, shrinking the linkedlist to only keep the recent one hour data.

For linkedlist, shrinking size could be done in O(1) time complexity by resetting the HEAD to a new data node. The discarded data nodes will be garbage-collected automatically.

But since we have a hashmap caching of these nodes, the data nodes discarded by the linkedlist are still referenced by the cache, which prevents GC from collecting them. So we will need to loop over them and remove each one of them from the hashmap, which takes O(n) time complexity where n is the numbers of discarded data nodes. It suddenly became the **bottleneck of performance**, not only because of the O(n) extra time cost, but more importantly, it will block the handling of newly arrived data nodes when we are shrinking the hashmap.

Can we do better?

## The Two Flashlight Paradox
Okay, I confess that I coined the phase *Two Flashlight Paradox*. It was just one scene from Stephen Chow's movie [From Beijing with Love(1994)](https://en.wikipedia.org/wiki/From_Beijing_with_Love).

In the movie, the rocket scientist Tat Man-sai invented a special flashlight that never get lit by itself until it senses light bean from another normal flashlight.
![img]({{ site.baseurl }}/public/flashlight0.jpg){: .center-image }*It won't light*
![img]({{ site.baseurl }}/public/flashlight1.jpg){: .center-image }*Until another flashlight lights*

It was a joke and a completely useless invention in the movie. But **Hey** isn't this what we want to solve our streaming data caching problem?

-- We want to keep data in hashmap(the joking flashlight to light) **only** when the data is referenced in the outside linkedlist(another flashlight).

## Introducing the Weakref.WeakValueDictionary class in Python.

Very well self-describing, the [WeakValueDictionary](https://docs.python.org/2/library/weakref.html#weakref.WeakValueDictionary) links its key, value pair using a "weak ref" instead of a strong ref(the normal ref) as the normal dictionary uses.
Like the joking flashlight in the movie, in WeakValueDictionary will keep its "key, value" pairs as long as the value is referenced by another strong ref outside of the WeakValueDictionary.

Once the outside strong ref is gone, both the value and the key in WeakValueDictionary will be removed immediately.

Let's do the following experiment to see how it works,

We will call the demo code below twice, 

In the first call, we feed it a normal Python dict() to do the cache.

In the second call, we feed it with a Weakref.WeakValueDictionary() to do the caching.

```python
import gc
from pprint import pprint
import weakref

gc.set_debug(gc.DEBUG_LEAK)

class ExpensiveObject(object):
    def __init__(self, name):
        self.name = name
        self.nxt = None
    def __repr__(self):
        return "ExpensiveObject(%s)" % self.name
    def __del__(self):
        print "(Deleting %s)" % self
        
def demo(cache_factory):

    dummyHead = ExpensiveObject("HEAD")
    cur = dummyHead
    print "CACHE TYPE: ", cache_factory
    cache = cache_factory() # the cache is initiated using cache_factory given
    for name in [ "one", "two", "three", "four" ]:
        o = ExpensiveObject(name)
        cache[name] = o
        cur.nxt = o
        cur = o
        del o # de-ref the o

    print "Before Shrinking, LinkedList contains:"
    p = dummyHead
    stringBuilder = []
    stringBuilder.append(p.name)
    p = p.nxt
    while (p):
        stringBuilder.append("->" + p.name)
        p = p.nxt
    print "".join(stringBuilder)

    print "Before Shrinking, cache contains:", cache.keys()
    for name, value in cache.items():
        print "  %s = %s" % (name, value)
        del value # decref

    # Shrink the LL
    print "Shrinking LL by reseting the head:"
    cur = dummyHead
    cur.nxt = cache["four"]
    # gc.collect() automatically

    print "After Shrinking, LinkedList contains:"
    p = dummyHead
    stringBuilder = []
    stringBuilder.append(p.name)
    p = p.nxt
    while (p):
        stringBuilder.append("->" + p.name)
        p = p.nxt
    print "".join(stringBuilder)


    print "After Shrinking, cache contains:", cache.keys()
    for name, value in cache.items():
        print "  %s = %s" % (name, value)

    print "Demo done. \nCleaning the following objects from the method stack."

print "\n##########Cachine Data Using Dict()############"
demo(dict)
print "\n##########Cachine Data Using WeakValueDictionary()############"
demo(weakref.WeakValueDictionary)

```

Let's check the output,

First for caching using Python's Dict().

```python
##########Cachine Data Using Dict()############
CACHE TYPE:  <type 'dict'>
Before Shrinking, LinkedList contains:
HEAD->one->two->three->four
Before Shrinking, cache contains: ['four', 'three', 'two', 'one']

Shrinking LL by reseting the head...Done

After Shrinking, LinkedList contains:
HEAD->four
After Shrinking, cache contains: ['four', 'three', 'two', 'one']

Demo done. 
Cleaning the following objects from the method stack.
(Deleting ExpensiveObject(HEAD))
(Deleting ExpensiveObject(one))
(Deleting ExpensiveObject(two))
(Deleting ExpensiveObject(three))
(Deleting ExpensiveObject(four))
```
From above, we can see even the linkedlist is cut short(the "one", "two", "three" nodes are removed from the LL), but discarded data nodes are still alive in memory, not being garbage-collected by Python's GC. That's because they are still being referenced by the keys in the cache. In this case, we will have to manually remove them from the cache.

Now let’s check the output for caching using Python’s WeakValueDictionary()


```python
##########Cachine Data Using WeakValueDictionary()############
CACHE TYPE:  weakref.WeakValueDictionary
Before Shrinking, LinkedList contains:
HEAD->one->two->three->four
Before Shrinking, cache contains: ['four', 'three', 'two', 'one']

Shrinking LL by reseting the head:
(Deleting ExpensiveObject(one))
(Deleting ExpensiveObject(two))
(Deleting ExpensiveObject(three))

After Shrinking, LinkedList contains:
HEAD->four
After Shrinking, cache contains: ['four']

Demo done. 
Cleaning the following objects from the method stack.
(Deleting ExpensiveObject(HEAD))
(Deleting ExpensiveObject(four))

```
From above, we can see the linkedlist is cut short, and discarded data nodes are immediately garbage-collected by Python's GC because the weakref.WeakValueDictionary() does not keep them once the strong reference is gong.

This way we do not need to do the cache shrinking manually. And the time complexity for cache shrinking is optimized to O(1).


## A concrete toy project example

In the link below, I solved an interesting logger system design problem using both ways above.
Check it out [here](http://weihan.online/blog/eugenejw.github.io/_site/2017/08/leetcode-359.html), if you are interetst.


