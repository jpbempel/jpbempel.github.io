---
title: "Yoda conditions"
layout: default
---
# Yoda conditions
This blog post is not about performance, but ranting. Ranting about too many developers that apply blindly some "rules" without knowing why there are those practices in the first place.
Recently, I have read this [article](http://blog.jooq.org/2015/08/11/top-10-useful-yet-paranoid-java-programming-techniques/) from Lukas Eder which in example number 4, recommends to inverse the natural way to write conditions by putting constant on left side of the operator instead of right side. The reason behind that is to avoid accidental assignment if you miss one equal sign.
```java
if (null == var1 && 0 == var2) {
  throw new IllegalArgumentException("invalid var1 & var2 value");
}
```
This practice (often called [Yoda Conditions](https://en.wikipedia.org/wiki/Yoda_conditions)) is quite old now. I encountered this the first time when doing C++. The language (in fact, this is C effectively) allows to do inline assignations inside a condition because conditions are in fact an expression evaluation (no boolean type in earlier version of C), and based on this evaluation if the result is 0 it means false, otherwise it is true. So any value (1, 2,3 -1, 0x012345f...) is considered as true.
Putting constant on the right side prevents (in case of a missing equal) the expression to compile so it helps to catch the mistake quickly.
If in C/C++ language this practice makes sense (discussing if it's is good or bad practice in C/C++ is out of the scope of this post) because there is rationale behind it.

In Java or C#, the story is totally different: Conditions are strongly typed to boolean. Assigning a reference to null or an integer variable inside a condition leads to compile error. Therefore, it does not make sense to follow this "rule" in those languages as the the mistake is inherently caught by the compiler thanks to the type system.

```java
if (var1 = null && var2 = 0) { // does not compile in Java
  throw new IllegalArgumentException("invalid var1 & var2 value");
}
```

Takeaway from this: Do not follow blindly "rules" without knowing their origin and what are they trying to address as issues. Know also your language and its type system. Rules like Yoda Conditions fall by themselves when leveraging correctly your language type system.
