---
layout: post
title: Java's Binary Search Madness
tags: [algorithms, bugs, jvm]
---
We all love [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm).
Straightforward, easy to understand algorithm; finds your number in a sorted array
in an appealing *O(log n)* time. But is it *that* simple to implement?

As it turns out, it's not. The way it was implemented in Java is
a great example of how a seemingly simple code should be treated with care. The bug
was introduced early in Java 1.2 but was fixed only for Java 1.6.

It was [Sedgewick's and Wayne's course on algorithms](http://algs4.cs.princeton.edu/home/) when
I've heard of this for the first time. Some time later I came over this
[Josh Bloch's post](https://research.googleblog.com/2006/06/extra-extra-read-all-about-it-nearly.html)
which has all essential information about the bug.

Let me guide you through this story once again, and share some trivia.

## The Bug and the Fix

This is how the binary search looks in Java 1.2 (JDK-1.2.2_007 sources):

{% highlight java linenos %}
    public static int binarySearch(long[] a, long key) {
	int low = 0;
	int high = a.length-1;

	while (low <= high) {
	    int mid =(low + high)/2;
	    long midVal = a[mid];

	    if (midVal < key)
		low = mid + 1;
	    else if (midVal > key)
		high = mid - 1;
	    else
		return mid; // key found
	}
	return -(low + 1);  // key not found.
    }
{% endhighlight %}
*Perfectionists, keep your anger on mixed tab/space indentation.*

So what's the deal? Well, simply finding midpoint between two numbers denoting
position in an array: <code>int mid =(low + high)/2;</code>. When <code>low + high</code>
exceeds max value of signed int (the only flavor you get in Java), it turns
into a negative value, and as a result, you get half of that negative value as 'mid'.

Here's a simple visualization of how it goes wrong:
{% highlight text %}
_ this is the sign bit
|
v
01010011 01110010 01001110 00000000 : 1400000000
    +
01010011 01110010 01001110 00000000 : 1500000000
    = (sum)
10101100 11011010 01111101 00000000 : -1394967296
    /= 2
11010110 01101101 00111110 10000000 : -697483648
{% endhighlight %}

There're several solutions, but the one applied in Java is simple and efficient - 
divide using bit shift operation, the *unsigned* one. As before, you get a negative
number as a result of overflow <code>(low + high)</code>, but the next operation
won't treat sign bit seriously, and will simply shift everything using zero as
a filler, thus getting us back on positive numbers' side:
{% highlight text %}
10101100 11011010 01111101 00000000 : -1394967296
    >>> 1
01010110 01101101 00111110 10000000 : 1450000000
{% endhighlight %}

## History and Trivia

The bug was identified in Tiger (1.5), and was finally fixed in two years for 1.6.
What I personally like most of this story are those comments under bug
[#5045882](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=5045582):

{% highlight text %}
EVALUATION

Finally fixing for Mustang.
"Can't even compute average of two ints" is pretty embarrassing.
                                     
2006-04-18
EVALUATION

Should be fixed in the next release. Not for Tiger.
###@###.### 2004-05-11
                                     
2004-05-11
{% endhighlight %}

End of story? No. The same problem hit again just around the time when binary
search seemed to have been fixed for Java 1.6
(see bug [#6437371](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6437371)),
this time in <code>TreeMap</code> which also uses midpoint calculation:
{%highlight text %}
Doug Lea writes:

"""
It just occurred to me that there is another binary-search-like indexing
overflow error, in TreeMap.java line 2357:
        int mid = (lo + hi) / 2;
ought to be
        int mid = (lo + hi) >>> 1;
"""
{% endhighlight %}

Now, end of story? Not quite. Let's pick some latest Java 9 source code, and
do some research. No fancy tools, no AST analysis - basic grep (assuming variable
names start with 'h' and 'l'). Out of 85 results, 25 survived manual review to
be presented below:
{%highlight bash %}
arthur@Arts-Mac:~/jdk-9/lib/src$ grep -r "\Wl\w* + h\w*" . | \
> awk '{print $1; $1=""; print "  " $0}'
./java.base/sun/util/calendar/ZoneInfo.java:
   int mid = (low + high) / 2;
./java.desktop/com/sun/media/sound/MidiUtils.java:
   ret = (low + high) >> 1;
./java.desktop/java/awt/font/NumericShaper.java:
   int mid = (lo + hi) / 2;
./java.desktop/sun/java2d/marlin/MergeSort.java:
   final int mid = (low + high) >> 1;
./java.desktop/sun/java2d/marlin/Renderer.java:
   mid = (low + high) >> 1;
./java.xml/com/sun/org/apache/xml/internal/dtm/ref/DTMDefaultBase.java:
   int mid = (low + high) / 2;
./java.xml/com/sun/org/apache/xml/internal/utils/NodeVector.java:
   int pivot = a[(lo + hi) / 2];
./java.xml/com/sun/org/apache/xml/internal/utils/NodeVector.java:
   a[(lo + hi) / 2] = a[hi];
./javafx.graphics/com/sun/marlin/DRenderer.java:
   mid = (low + high) >> 1;
./javafx.graphics/com/sun/marlin/DRendererNoAA.java:
   mid = (low + high) >> 1;
./javafx.graphics/com/sun/marlin/MergeSort.java:
   final int mid = (low + high) >> 1;
./javafx.graphics/com/sun/marlin/Renderer.java:
   mid = (low + high) >> 1;
./javafx.graphics/com/sun/marlin/RendererNoAA.java:
   mid = (low + high) >> 1;
./jdk.compiler/com/sun/tools/javac/util/Position.java:
   int mid = (low + high) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/debugger/bsd/BsdCDebugger.java:
   mid = (low + high) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/debugger/cdbg/basic/BasicCDebugInfoDataBase.java:
   int midIdx = (lowIdx + highIdx) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/debugger/cdbg/basic/BasicLineNumberMapping.java:
   int midIdx = (lowIdx + highIdx) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/debugger/proc/ProcCDebugger.java:
   mid = (low + high) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/debugger/windbg/DLL.java:
   int curIdx = ((loIdx + hiIdx) >> 1);
./jdk.hotspot.agent/sun/jvm/hotspot/oops/GenerateOopMap.java:
   int m = (lo + hi) / 2;
./jdk.hotspot.agent/sun/jvm/hotspot/oops/InstanceKlass.java:
   int mid = (l + h) >> 1;
./jdk.hotspot.agent/sun/jvm/hotspot/utilities/SystemDictionaryHelper.java:
   mid = (low + high) >> 1;
./jdk.jconsole/sun/tools/jconsole/inspector/TableSorter.java:
   mid = getValueAt( ( lo0 + hi0 ) / 2 , key);
./jdk.scripting.nashorn/jdk/nashorn/internal/runtime/regexp/joni/CodeRangeBuffer.java:
   final int x = (low + high) >> 1;
./jdk.scripting.nashorn/jdk/nashorn/internal/runtime/regexp/joni/EncodingHelper.java:
   final int x = (low + high) >> 1;
{% endhighlight %}

## Conclusions
If you write tests, do boundary testing.