---
layout: post
title:  "Null, y u so ambiguous!"
date:   2015-01-26 19:04:39
categories: code
---

I've been reading through the [Guava library docs][guava] and found this interesting.

Try not to use null.  What does null mean really?  Does it mean success?  Does it mean failure?  Does a null from map.get(key) mean that the value in the map is null or that the value doesn't exist in the map?  It is ambiguous and a major cause of bugs in larger systems.

Optional<T> is a way of replacing a nullable T reference with a non-null value. An Optional may either contain a non-null T reference (in which case we say the reference is "present"), or it may contain nothing (in which case we say the reference is "absent"). It is never said to "contain null."

{% highlight java %}
Optional<Integer> possible = Optional.of(5);
possible.isPresent(); // returns true
possible.get(); // returns 5
{% endhighlight %}

Besides the increase in readability that comes from giving null a name, the biggest advantage of Optional is its idiot-proof-ness. It forces you to actively think about the absent case if you want your program to compile at all, since you have to actively unwrap the Optional and address that case. Null makes it disturbingly easy to simply forget things, and though FindBugs helps, we don't think it addresses the issue nearly as well.

[guava]: https://code.google.com/p/guava-libraries/wiki/UsingAndAvoidingNullExplained