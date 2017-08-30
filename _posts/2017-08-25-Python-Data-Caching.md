---
layout: post
title: Python GC, memory leak and one example of how to take full advantage of it.
---


<div class="message">
  Howdy! This is an unfinished post! I am still working on it. 
</div>


When switched from the language with manual memory management, such as C or C++, to a garbage-collected language, your job as a programmer is made much easier by the fact that your objects are automatically reclaimed when you’re through with them. It seems almost like magic when you first experience it, it can easily lead to the impression that you don’t have to think about memory management, but this isn’t quite true.

Ignoring memory management will lead you to situations where thing can go very wrong, for example, passing a recall method to an object, which does not explicitly destroy (by nulling it out) it when it became no longer needed. Overtime, it will result in memory leak problems. (Commonly seen in Observer & Observable Models)

Keeping memory management in mind will also help you gain suprising edges when dealing with data caching tasks.
Today I am going to show you how to take advantage of GC in Python (may apply to Java as well).





## The problem to solve
Design a logger system that receive stream of messages along with its timestamps, each message should be printed if and only if it is not printed in the last 10 seconds.

Given a message and a timestamp (in seconds granularity), return true if the message should be printed in the given timestamp, otherwise returns false.

## The naive solution

the algorithm is described as below,
Initiate a hashmap, where the key is the log content and the value is the log's timestamp.
for every new incoming log, we check if it is already in the hashmap.
   if no, we know it is a newly appeared log. we print it and store it into the hashmmap.
   if yes, we check the timestamp of the log in hashmap. 
      if the time difference is greater than 10-min, we print the log, and do an update to set the value to current timestamp.
      if the time difference is smaller than 10-min, we do not print the log, and do nothing.

This data structure works well if you are lucky that most of the logs you recieved are having the same content, so that the hashtable won't grow big.

When the hashmap grow big the collision will happen, and it starts to degereate to linkedlists (or BSTs in Java 8).

## The memory concerning solution
So ideally, we want to limit the size of the hashmap.
In other word, the problem is demanding a data structure that,
1. we maintain a timeline linkedlist that all logs are wrapped as object and get stored in chronical order. 
2. we also maintain a hashmap where the key is the log content, the value is a ref to the log object stored in the linkedlist.
3. for every new incoming log, we need to check whether it is in the timeline list by looking up the hashmap in O(1) time complexity.
   If no, print it and add it to the end of the timeline list.
   If yes, we check the timestamp difference,
      if it is smaller than 10-min, we do not print the newly received log, and do nothing.
      if it is greater than 10-min, we do two things -- 
      	 1) all logs before the existinglog (including it) are older than 10-min, we remove them from the timeline list by setting the linkedlist HEAD to the existing log's next log object in O(1) time complexity.
	    and we shrink the hashmap by removing all the older logs's reference out from the hashmap in O(n) time complexity, where n is the size of the current hashmap.
	 2) we print the log and add it to the end of the timeline list.

This solution is basically avoid store the full logs, which most of them could be useless.

The performance bottleneck is the hashmap shrink, which is of O(n) and blocks the incoming log handling.

## The memory concerning solution and O(1) time comlexity solution
The new algorithm is the same as the 2nd solution. The only difference is we use a WeakValueDictionary to do the cache instead of using a regular hashmap.

The [WeakValueDictionary](https://docs.python.org/2/library/weakref.html#weakref.WeakValueDictionary) links its key, value pair using a "weak ref" instead of a strong ref(the normal ref) as the dictionary uses.
In WeakValueDictionary, as long as the value is referenced by another strong ref, the WeakValueDictionary will key the value. Once the strong ref is gone, both the value and the key in WeakValueDictionary will be gone.

Let's do the following experiment,
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
        return 'ExpensiveObject(%s)' % self.name
    def __del__(self):
        print '(Deleting %s)' % self
        
def demo(cache_factory):
    # hold objects so any weak references 
    # are not removed immediately

    # the cache using the factory we're given
    dummyHead = ExpensiveObject("HEAD")
    cur = dummyHead
    print 'CACHE TYPE:', cache_factory
    cache = cache_factory()
    for name in [ 'one', 'two', 'three', 'four' ]:
        o = ExpensiveObject(name)
        cache[name] = o
        cur.nxt = o
        cur = o
        del o
        
        #all_refs[name] = o
        #del o # decref

    #print 'all_refs =',
    #pprint(all_refs)
    print 'Before, cache contains:', cache.keys()
    for name, value in cache.items():
        print '  %s = %s' % (name, value)
        del value # decref
        
    # Remove all references to our objects except the cache
    print 'Cleanup:'
    cur = dummyHead
    cur.nxt = cache['four']
    gc.collect()

    print 'After, cache contains:', cache.keys()
    for name, value in cache.items():
        print '  %s = %s' % (name, value)
    print 'demo returning'
    return

demo(dict)
print
demo(weakref.WeakValueDictionary)

```

The output,
```python
python weak_ref3.py
CACHE TYPE: <type 'dict'>
Before, cache contains: ['four', 'three', 'two', 'one']
  four = ExpensiveObject(four)
  three = ExpensiveObject(three)
  two = ExpensiveObject(two)
  one = ExpensiveObject(one)
Cleanup:
After, cache contains: ['four', 'three', 'two', 'one']
  four = ExpensiveObject(four)
  three = ExpensiveObject(three)
  two = ExpensiveObject(two)
  one = ExpensiveObject(one)
demo returning
(Deleting ExpensiveObject(HEAD))
(Deleting ExpensiveObject(one))
(Deleting ExpensiveObject(two))
(Deleting ExpensiveObject(three))
(Deleting ExpensiveObject(four))

CACHE TYPE: weakref.WeakValueDictionary
Before, cache contains: ['four', 'three', 'two', 'one']
  four = ExpensiveObject(four)
  three = ExpensiveObject(three)
  two = ExpensiveObject(two)
  one = ExpensiveObject(one)
Cleanup:
(Deleting ExpensiveObject(one))
(Deleting ExpensiveObject(two))
(Deleting ExpensiveObject(three))
After, cache contains: ['four']
  four = ExpensiveObject(four)
demo returning
(Deleting ExpensiveObject(HEAD))
(Deleting ExpensiveObject(four))
root@localhost:~/my_code# 
```



story starts at one easy yet worth digging problem posted on Leetcode -- 

The problem is easy to solve by storing all log received in a hashmap, where the key is the log content and value is the log's timestamp.


until you start to worry about the glowing size of the hashmap.

The ideal solution is to maintain only the most recent 10-minute windows of logs in the hashmap, and get rid of the older one.
Below is my implementation of the [solution](http://weihan.online/blog/eugenejw.github.io/_site/2017/08/leetcode-359.html)

I used a singly linkedList to link all logs by the order of time when it arrived.
Let's say so far we have received four logs as below
```java
LinkList data structure:
node('Sam Logged in at 12:00')@address200 -> node('Tom Logged in at 12:06')@address201 -> node('Julian Logged in at 12:07')@address7896 -> node('Sam Logged in at 12:11')@139 -> null
```
And we store them into a hashmap where the key is the log's content, and the value is the RAM address of its position in linkedlist above.
```java
HashMap data structure:
{
node('Sam Logged in'): (@address200, 12:00), 
node('Tom Logged in'): (@address201, 12:06), 
node('Julian Logged in'): (@address7896, 12:07)
}
```

so when the objects in linkedlist is gone, the wkey, value pairs in eakrefvaluedictionary are gone too.









Cum sociis natoque penatibus et magnis <a href="#">dis parturient montes</a>, nascetur ridiculus mus. *Aenean eu leo quam.* Pellentesque ornare sem lacinia quam venenatis vestibulum. Sed posuere consectetur est at lobortis. Cras mattis consectetur purus sit amet fermentum.

> Curabitur blandit tempus porttitor. Nullam quis risus eget urna mollis ornare vel eu leo. Nullam id dolor id nibh ultricies vehicula ut id elit.

Etiam porta **sem malesuada magna** mollis euismod. Cras mattis consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla sed consectetur.

## Inline HTML elements

HTML defines a long list of available inline tags, a complete list of which can be found on the [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).

- **To bold text**, use `<strong>`.
- *To italicize text*, use `<em>`.
- Abbreviations, like <abbr title="HyperText Markup Langage">HTML</abbr> should use `<abbr>`, with an optional `title` attribute for the full phrase.
- Citations, like <cite>&mdash; Mark otto</cite>, should use `<cite>`.
- <del>Deleted</del> text should use `<del>` and <ins>inserted</ins> text should use `<ins>`.
- Superscript <sup>text</sup> uses `<sup>` and subscript <sub>text</sub> uses `<sub>`.

Most of these elements are styled by browsers with few modifications on our part.

## Heading

Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor. Duis mollis, est non commodo luctus, nisi erat porttitor ligula, eget lacinia odio sem nec elit. Morbi leo risus, porta ac consectetur ac, vestibulum at eros.
### Strong Ref

```python
import time
import os
from random import randint
import weakref

class Observable:
    def __init__(self):
        self.__observers = []

    def register_observer(self, observer):
        self.__observers.append(observer)

    def notify_observers(self, *args, **kwargs):
        for observer in self.__observers:
            observer.notify(self, *args, **kwargs)

class Observer:
    def __init__(self):
        self._cahce = []
        for _ in xrange(100000):
            self._cahce.append(randint(1, 10000000))

    def notify(self, observable, *args, **kwargs):
        print('Got', args, kwargs, 'From', observable)


observables = [Observable() for i in range(10000)]
for observable in observables:
    observer = Observer()
    #print "Observable subject: {}; Observer obj: {}".format(id(subject), id(observer))
    observable.register_observer(observer)
    time.sleep(3)
    #observable.notify_observers('test')
    os.system('free -m |awk \'{print $4}\' |sed -n \'2p\'')

```


### Weak Ref

```python
import time
import os
from random import randint
import weakref

class Observable:
    def __init__(self):
        self.__observers = []

    def register_observer(self, observer):
        self.__observers.append(observer)

    def notify_observers(self, *args, **kwargs):
        for observer in self.__observers:
            observer.notify(self, *args, **kwargs)

class Observer:
    def __init__(self):
        self._cahce = []
        for _ in xrange(100000):
            self._cahce.append(randint(1, 10000000))

    def notify(self, observable, *args, **kwargs):
        print('Got', args, kwargs, 'From', observable)


observables = [Observable() for i in range(10000)]
for observable in observables:
    observer = Observer()
    weakRefObserver = weakref.ref(observer)
    #print "Observable subject: {}; Observer obj: {}".format(id(subject), id(observer))
    observable.register_observer(weakRefObserver())
    time.sleep(3)
    #observable.notify_observers('test')
    os.system('free -m |awk \'{print $4}\' |sed -n \'2p\'')

```




Cum sociis natoque penatibus et magnis dis `code element` montes, nascetur ridiculus mus.

{% highlight js %}
// Example can be run directly in your JavaScript console

// Create a function that takes two arguments and returns the sum of those arguments
var adder = new Function("a", "b", "return a + b");

// Call the function
adder(2, 6);
// > 8
{% endhighlight %}

Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa.

### Lists

Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.

* Praesent commodo cursus magna, vel scelerisque nisl consectetur et.
* Donec id elit non mi porta gravida at eget metus.
* Nulla vitae elit libero, a pharetra augue.

Donec ullamcorper nulla non metus auctor fringilla. Nulla vitae elit libero, a pharetra augue.

1. Vestibulum id ligula porta felis euismod semper.
2. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus.
3. Maecenas sed diam eget risus varius blandit sit amet non magna.

Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.

<dl>
  <dt>HyperText Markup Language (HTML)</dt>
  <dd>The language used to describe and define the content of a Web page</dd>

  <dt>Cascading Style Sheets (CSS)</dt>
  <dd>Used to describe the appearance of Web content</dd>

  <dt>JavaScript (JS)</dt>
  <dd>The programming language used to build advanced Web sites and applications</dd>
</dl>

Integer posuere erat a ante venenatis dapibus posuere velit aliquet. Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Nullam quis risus eget urna mollis ornare vel eu leo.

### Tables

Aenean lacinia bibendum nulla sed consectetur. Lorem ipsum dolor sit amet, consectetur adipiscing elit.

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Upvotes</th>
      <th>Downvotes</th>
    </tr>
  </thead>
  <tfoot>
    <tr>
      <td>Totals</td>
      <td>21</td>
      <td>23</td>
    </tr>
  </tfoot>
  <tbody>
    <tr>
      <td>Alice</td>
      <td>10</td>
      <td>11</td>
    </tr>
    <tr>
      <td>Bob</td>
      <td>4</td>
      <td>3</td>
    </tr>
    <tr>
      <td>Charlie</td>
      <td>7</td>
      <td>9</td>
    </tr>
  </tbody>
</table>

Nullam id dolor id nibh ultricies vehicula ut id elit. Sed posuere consectetur est at lobortis. Nullam quis risus eget urna mollis ornare vel eu leo.

-----

Want to see something else added? <a href="https://github.com/poole/poole/issues/new">Open an issue.</a>
