---
layout: post
title: Narrowing gets funny
---

Write accurate calculation code requires to know about the border line case.
The following code
{% highlight java %}
int intValue = (int) Float.NEGATIVE_INFINITY;
System.out.println(intValue);
{% endhighlight %}
prints something like this:
{% highlight java %}
-2147483648
{% endhighlight %}

The conversion to integers returns the lowest value of the primitive type. Narrowing from float to long works the same. 
Up to now, everything's ok. What about smaller primitive type such as byte, char and short ?

{% highlight java %}
short shortValue = (short) Float.NEGATIVE_INFINITY;
System.out.println(shortValue);</code>
{% endhighlight %}
Output:
{% highlight java %}
0
{% endhighlight %}

Outch. The [Java specification]({% "http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.3 %}) 
reveals that the float is converted to an int, and clamped to the negative value of largest magnitude: `0x80000000`. 
Then the 32 bit binary representation is truncated to a 16 bit short: `0x0000`.

It is possible compute the largest float value that is converted without causing overflow:
{% highlight java %}
Float.intBitsToFloat(Float.floatToIntBits(0x8000)-1);
{% endhighlight %}

Let it be said that developers never finish learning.
