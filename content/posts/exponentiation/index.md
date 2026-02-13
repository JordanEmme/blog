+++
title = 'Fast Exponentiation and Floating Point Representations'
date = '2026-02-13'
draft = true
toc = true
tocBorder = true
math = true
[params]
  author = "Jordan Emme"
+++

Where I briefly explain an old school trick to approximate the exponential function, by abusing the
way floating points are represented in memory. The method is known since at least 1999, where it
appears in a [paper by Nicol Schraudolph](https://nic.schraudolph.org/pubs/Schraudolph99.pdf).

## Floating Point Representation

Floating points are absolutely ubiquitous in programming these days, and virtual every architecture
implements the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard. Paradoxically, it seems
few software engineers are aware of the definition and subtleties of the representation, and even
less so of the sort of abuse they lend themselves to.

An almost mythical example of such hacks is the (misattributed) Quake III fast inverse square root
algorithm:

```c
float Q_rsqrt( float number )
{
    long i;
    float x2, y;
    const float threehalfs = 1.5F;
    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

  	return y;
}  
```

There is not anything I can say about it that has not been extensively said already and isn't
documented and linked from the Wikipedia page, so I won't waste any time on that and focus on the
exponential instead. By the end, that should still demystify the line:

```c
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
```

### The IEEE 754 Standard

Although there has historically been many ways to represent floating points (different bases, with
or without gradual underflow etc...), we have eventually settled on a standard that I'll explain.
Unless you code on some pretty old/esoteric machine, or do some AI stuff that require the usage of
more exotic formats, chances is are this is what you're dealing with.

We'll focus on 32 bits floating points (float in C/C++) here, but the standard is similarly defined
for 16 bits and 64 bits floating points, just with a couple different constants and bit partition.

Let \\(f\\) be a 32 bits floating point. Its binary expression is partitioned as follows:
|s|e7|e6|e5|e4|e3|e2|e1|e0|m22|m21|...|m1|m0|
|-|--|--|--|--|--|--|--|--|---|---|---|--|--|

Let
\\[
e = \sum_{i = 0}^{7} e_i 2^i \quad\text{and}\quad m =  \sum_{i = 0}^{22} m_i 2^i,
\\]

then, 

\\[
f = (-1)^s \times 2^{e - 127} \times \left(1 + 2^{-23}m\right).
\\]

> __Remark__: There may be something surprising about having the exponent being the biased integer
\\(e - 127\\) and using the binary representation of the exponent as an unsigned integer, rather
than doing the usual "ones complement" representation to have a signed integer instead, but this
is for good reasons as it allows the ordering of floating points to coincide with the lexicographic
ordering of their binary representation.

There are some subtleties here, as the previous formula does not hold for \\(e = 0\\)
or \\(e = 255\\), which allows on the one hand for gradual underflow with [subnormal
numbers](https://en.wikipedia.org/wiki/Subnormal_number) and for having dedicated representations
of infinity and NaN on the other hand. It's good to be aware of it to have the full picture, but we
won't need that here and we ignore edge cases for simplicity.

## The Algorithm

### Code

Now we know what floating points look like under the hood, let's see how to abuse them to
approximate the exponential. Let's start with a somewhat less hacky way to do type punning than is
used in the fast inverse square root algorithm, by defining the following union:

```c
typedef union F32 {
    float value;
    uint32_t bits;
} F32;
```

The C++ masochists are welcome to `reinterpret_cast` or `bit_cast` to their heart's content instead.

With that, we can now also write a seemingly undecipherable algorithm to approximate the exponential

```c
float fast_exp(float f) {
    float fb = f * 12102203;
    F32 expo = {.bits = (int)fb + 1064986823};
    return expo.value;
}
```

### Explanation

There are only three lines, and essentially three things to explain in that piece of code:

1. What is the constant \\(12102203\\)?
1. What does the cast to `int`, then the reinterpretation of the bits to a floating point achieve?
1. What is the constant \\(1064986823\\)?

#### The Constant 12102203

Why 12102203? Well, it turns out this is the value of \\(\text{log}_2(e) \times 2^{23}\\). Notice
this is an integer because we have 23 bits of precision with 32bits floating points.

#### The Constant 1064986823

This one is more tricky. "Half" of it is straightforward, and the other one is an optimisation.
The constant 1064986823 can be written as:
\\[
1064986823 = (127 \ll 23) - 366393.
\\]

Here, 127 is the bias of 32-bit floating points, it is shifted left 23 times (or multiplied by
\\(2^{23}\\)), and 366393 is an optimisation constant computed to minimise the maximum relative
error between our approximation of exponential and the hardware returned value of the exponential
function.

#### Cast and Bit/Reinterpret Cast

Perhaps the most interesting, and most illuminating part, is realising that reinterpreting a
(non-negative) signed integer as a floating point is akin to applying the function \\(x \mapsto
2^{x-127}\\), and thus, the exponential. Let's make this a bit more precise.

Let \\(x\\) be a non-negative integer, with binary representation \\(x_{30}...x_{0}\\) (\\(x_{31} =
0\\) since the integer is non-negative). Reinterpreting \\(x\\) bits as a float yields the floating
point
\\[
x_f = 2^{(x \gg 23) - 127} \left( 1 + \sum_{i=0}^{22}x_i2^{i-23}\right).
\\]

#### Making Sense of it All

Given the previous expression of \\(x_f\\), the goal is the following: for a given floating point
\\(f\\) that we wish to get the exponential of, we try and compute an integer \\(f_b\\) such that
\\(2^{(f_b \gg 23) - 127}\\) is as close as possible to \\(e^f\\).

First, remark that
\\[
2^{x\\, \text{log}_2(e)} = e^{x\\, \text{log}_2(e)\text{ln}(2)}, 
\\]
so
\\[
2^{x\\, \text{log}_2(e)} = e^{x\\, \text{ln}(e)}, 
\\]
which implies 
\\[
2^{x\\, \text{log}_2(e)} = e^{x}. 
\\]

Thus, we want \\((f_b \gg 23) - 127\\) close to \\(f \times \text{log}_2(e)\\). Let's derive this:

\\[
(f_b \gg 23) - 127 \simeq f \times \text{log}_2(e)
\\]
is equivalent to, by multiplying each side by \\(2^{23}\\)
\\[
f_b - 127 \times 2^{23} \simeq f \times \text{log}_2(e) \times 2^{23}
\\]
and we get
\\[
f_b \simeq f \times \text{log}_2(e) \times 2^{23} + 127 \times 2^{23}.
\\]
We should recognise here (part of) the constants which were necessary to readjust the
exponentiation induced by the bit-cast of an integer to a floating point.

This kind of computation can be extended to understand what happens between the
\\(2^i\\) and \\(2^{i + i}\\) images --- spoilers: it's a piecewise affine function
--- as well as doing a rigorous analysis of the errors. This is done in the [original
paper](https://nic.schraudolph.org/pubs/Schraudolph99.pdf) I mentioned at the very start.

## Conclusion

So there you have it. Taking advantage of the hardware representation of floating points, it
is possible to do a (*a priori*) fast approximation of the exponential function, with only one
multiplication, one cast to `int`, and one addition.

If you were to check the relative error on the entire 32-bit floating points range (with some
attention about possible edge cases with 0 and INF/NaN), you'll find it to be bounded by around
3%. This is not bad for what little work our function does but can be improved with further
adjustments... at the cost of performance of course.
