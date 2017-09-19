---
layout: post
title: Bloom Filter -- a probabilistic data structure as alternative to HashSet
---

As a software engineer, I trust the HashSet. If the HashSet tells me that "I defintely do not contains this key", I take its words, unless, well my RAM gets bit alterations from "Atmospheric Neutrons" (aka: Cosmic Rays).

As an alternative to hashset, the Bloom Filter is a similar data structure that does the same job, but in a more space-efficient manner. 

## Motivation -- Malicious Website Blocking Problem

Let's say we use crawlers to scan websites's content to ensure it is not comprimised by malware. When a certain url is believed to be comprised, we add the url to a black list. As the Google Chrome browser does.

![img]({{ site.baseurl }}/public/bloom_filter.jpg){: .center-image }*Chrome browser is using bloom filter to block malicious websites*

We can implement this blacklist using a hashtable. This table could be located either on client side(i.e. the Chrome browser) or on the server side(i.e. online shorten url service provider). This approach is conceptually easy to understand, simple to implement with lines of code, and with O(1) time complexity for url look-ups. 

But with the following drawbacks,

* Not memory efficient -- for hashset containing top 1 million alexa domains, the hashset size is roughly 300M in memory. For shorten url provider, like [bitly](https://bitly.com/), the blacklist could take about 2.9GB in memory (according to its blog post).
* We don't want to expose the raw data to client side -- the blacklist itself is a precious assets, you don't want to expose it to hacker.

With the Bloom Filter, the 300MB hashtable could be reduced to 1.71MB with 0.001 false positive rate, calculated by [bloom filter calculater](https://hur.st/bloomfilter?n=1000000&p=0.001).

And because, ubder the hood, the bloom filter is a bitarray with '1's and '0's, it encripted the actual content of the hashtable.

## Introducing Bloom Filter

> A Bloom filter is a space-efficient probabilistic data structure, conceived by Burton Howard Bloom in 1970, that is used to test whether an element is a member of a set. False positive matches are possible, but false negatives are not – in other words, a query returns either "possibly in set" or "definitely not in set". Elements can be added to the set, but not removed (though this can be addressed with a "counting" filter); the more elements that are added to the set, the larger the probability of false positives.

Quote from [Bloom Filter, Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter)

Bloom Filter is a bit array of N bits, where N is the size of the bit array. It has another parameter which is the number of hash functions, k. These hash functions are used to set bits in the bit array. When inserting an element x into the filter, the bits in the k indices h1(x), h2(x), ..., hk(x) are set, where the bit positions are determined by the hash functions. Note that as we increase the number of hash functions, the false positive rate of this probability goes to zero. However, it takes more time to insert and lookup as well as the bloom filter fills up more quickly.

In order to to membership existence in the Bloom Filter, we need to chekck if all of the bits are set; very similar to how we insert item into a bloom filter. If all of the bits are set, then it means that that item is probably in the bloom filter, where if anot all of the bits are set, then it means that the item is not in the Bloom Filter.

Check this link for detailed [tutorial for bloom filter](https://llimllib.github.io/bloomfilter-tutorial/)

## An example bloom filter containing 1-million urls

For a bloom filter containing 1 million urls, with 0.001 false positive rate.

* If the bloom filter tells you "it is definitely NOT in the bloom filter", it is 100% ensured that it is not in the bloom filter (because of the 0 false nagative rate guarantee).
* If the bloom filter tells you "It might be in the bloom filter", there is a false positive rate of 0.001.

## What is positives matter?

A false positive error, or in short a false positive, commonly called a "false alarm", is a result that indicates a given condition exists, when it does not. 

For example, if the bloom filter says the "www.facebook.com" is NOT in the blacklist (abvious, it should not), you can give this url a greenlight to let it pass.
If the bloom filter says the "www.youtube.com" might be in the blacklist, you take this with **a grain of salt** -- bearing in mind that there is 0.1% of false positive. So you need send the url "www.youtube.com" to further examinations, for example, a DB check, or even some more expensive realtime malware scan. 

If your further check confirms thata the "www.youtube.com" is **NOT** a malicious website, it is a "false positive" or "false alarm". If, somehow, "www.youtube.com" is marked as a malicous website in your DB. It is a "true positive".

## Does false positive matters? 

In this case, the false positive is acceptable, as long as we control the rate to be low by allocating more space to the bloom filter.

Indeed, to handle the occasional false positives, we did spend more time and resource to do the checking in the backend. 

It is a trade-off decision to make. More importantly, not a single malicious website gets **slip out of our hands**, because of the "no false negative" guarantee.

## Implementing the bloom filter above

Without concerning the thread safe, entry-removing needs, or scalability needs, the bloom filter could be achieved in a few lines of code.

Below is the python 2.7 implementation of a bloom filter containing 1 million url, with false positive rate set to 0.001.
It uses the mmh3 -- [MurmurHash3](https://en.wikipedia.org/wiki/MurmurHash) hash function, and by feeding the mmh3 with different seeds, they can be treated as different hashfunctions.
 
```python
import mmh3
import sys
import time
from bitarray import bitarray
import csv

class BloomFilter(set):

    def __init__(self, size, hash_count):
        super(BloomFilter, self).__init__()
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)
        self.size = size
        self.hash_count = hash_count

    def __len__(self):
        return self.size

    def __iter__(self):
        return iter(self.bit_array)

    def add(self, item):
        for ii in range(self.hash_count):
            index = mmh3.hash(item, ii) % self.size
            self.bit_array[index] = 1

        return self

    def __contains__(self, item):
        out = True
        for ii in range(self.hash_count):
            index = mmh3.hash(item, ii) % self.size
            if self.bit_array[index] == 0:
                out = False

        return out

class HashSetLookup(object):
    def __init__(self):
        self._hs = set()

    def add(self, word):
        self._hs.add(word)

    def check(self, word):
        if word in self._hs:
            return True
        else:
            return False

bf = BloomFilter(14377588, 10) #parameters generated from the bloom filter calculater
hs = HashSetLookup()

count = 0
for row in csv.reader(open("top1m.csv")):
    bf.add(row[1])
    hs.add(row[1])
    count += 1
print "{} items added!".format(count)

print "Size of the Bloom Filter in MB: ", 14377588*0.6/100000
print "Lookup using Bloom Filter..."
start = time.time()
not_seen_count = 0
for row in csv.reader(open("2m_incoming_traffic.csv")):
    #print "look up:", row[1]                                                                                                     
    if row[1] not in bf:
        not_seen_count += 1
    #    if ".dummy" not in row[1]:                                                                                               
    #        print row[1]                     

print "False Positive count: ", not_seen_count

end = time.time()
print "time elapsed for 2m incoming url traffic: ", (end - start), "s"
print "not seen url count: ", not_seen_count
print "false positive count: ", 999200 - not_seen_count

print "Size of the hashset in MB: ", sys.getsizeof(hs._hs)/100000
print "Lookup using hashSet..."
start = time.time()
not_seen_count = 0
for row in csv.reader(open("2m_incoming_traffic.csv")):
    #print "look up:", row[1]                                                                                                     
    if not hs.check(row[1]):
        not_seen_count += 1
end = time.time()
print "time elapsed for 2m incoming url traffic: ", (end - start), "s"
print "not seem url count: ", not_seen_count

```

Output, 

```python
mbp:python-bloomfilter weihan$ python bloom_filter.py
999200 items added!

##########lookup 2-million url via bloom filter#########
Size of the Bloom Filter in MB:  86.265528
Lookup using Bloom Filter...
time elapsed for 2m incoming url traffic:  12.2198560238 s
not seen url count:  998209
false positive count:  991

##########lookup 2-million url via hashset#########
Size of the hashset in MB:  335
Lookup using hashSet...
time elapsed for 2m incoming url traffic:  1.70107579231 s
not seem url count:  999200
```

From the output above we can observe the 3 facts,

* There are 991 false positive cases reported. 
* The bloom filter is of sze 86MB, while the hashset with the same contents cost 335MB in RAM.
* The bloom filter cost more time to do the lookup -- because hashset uses only one hashfunction; while bloom filter utilizes multiple hash functions.


## full fledged implementations in different languages,


In production, for different reasons, you might want to write your own bloom filter implementation to fully fit your needs, for example, speed requirent, distributed deployment, or removal requirements. 

There are open source implementations for every language, but the following implementations are pretty good in my experience:

* [Python scalable implementation](https://github.com/jaybaird/python-bloomfilter)
* [Java thread safe implementation](https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/BloomFilter.java)
* [Python scalable & removal-supported implementation](https://github.com/bitly/dablooms)

Note: you can re-write the python code using pypy, if speed is crucial in your application.


