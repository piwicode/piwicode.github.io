---
layout: post
title: Narrowing gets funny
excerpt: Conversion from floating point to integer is error-prone
---

Write accurate calculation code requires to know about the edge cases.
Let's explore what happens when casting a `float` into an `int`, in particular when the values overflow.

{% highlight java %}
int intValue = (int) Float.NEGATIVE_INFINITY;
System.out.println(intValue);
// => -2147483648
{% endhighlight %}

The conversion to integers returns the closest value below lowest value of the primitive type. Narrowing from float to long works the same.
There is nothing surprising about this, but what about smaller primitive types such as `byte`, `char` and `short` ?

{% highlight java %}
short shortValue = (short) Float.NEGATIVE_INFINITY;
System.out.println(shortValue);
// => 0
{% endhighlight %}

Outch. The [Java specification](http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.3)
reveals that the float is converted to an `int`, and clamped to the negative value of largest magnitude: `0x80000000`.
Then the 32 bit binary representation is truncated to a 16 bit `short`: `0x0000`.

It is possible compute the float range that is converted without causing overflow:
{% highlight java %}
Float.intBitsToFloat(Float.floatToIntBits(Short.MAX_VALUE+1)-1);
// => 32767.998
Float.intBitsToFloat(Float.floatToIntBits(Short.MIX_VALUE-1)-1);
// => -32768.996
{% endhighlight %}

The trick consists in selecting the first integer that cause an overflow by explicitly going over the `short` range boundaries. This is what `Short.MAX_VALUE + 1` achieves.

Then the integer value is converted to `float`. This is an exact conversion given that `float` has a 23 bits mantissa. This `float` value is the smallest one that causes an overflow.

Finally the next float `value` toward zero is selected. As a consequence the rounding toward zero will match the right boundary of short value interval.

Everyday is a school day.
