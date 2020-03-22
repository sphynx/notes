---
description: Help! My printf is producing digits out of thin air!
---

# printf\(\) and floating point numbers

One day we had certain mismatch between floating point numbers. One number when inspected in an IDE looked much longer than the other, having lots of extra digits. Then a colleague of mine said that it's fine, they might still be the same number and produced some code similar to this:

```c
#include <stdio.h>

int main(void)
{
    double d = 0.1234567890123456;  // 16 digits 
    printf("%.16f\n", d);
    printf("%.55f\n", d);
    return 0;
}
```

What do you think it will print? Most programmers know that double precision has about 16 significant decimal digits when numbers are in that range \(i.e between 0 and 1\). So I am printing here 16 digits first and then some more just to see what comes next. It prints this:

```c
> gcc 1.c && ./a.out
0.1234567890123456
0.1234567890123455941031593852130754385143518447875976562
```

This looked like quite a lot of extra digits and it did not even stop there! So I tried to understand what's going on.

I know that we represent real decimal numbers with bits stored according to IEEE 754 floating point standard, so our decimal literal is somewhat imprecisely stored in binary representation. Now it looks like `printf` is printing that decimal representation. However, what was surprising to me is that this imprecise representation can be expanded to so many new decimal digits.

Let's figure out step by step what's happening in this little program. The interesting steps with accompanying questions I had are below:

1. Compiler has to convert that string representing a decimal C-literal into a double. How that should be done? Are there restrictions on the literal length?
2. That double is represented with some bits. What bits exactly? How they are laid out in memory? How can I build them by hand?
3. Those bits can be converted back to decimal and printed with `printf`. How many digits can I expect in this decimal expansion?

Let's first address the representation question, then we'll have terminology to discuss parsing of the literals and finally converting back to decimal with `printf`.

### Representing doubles with IEEE 754

I assume that you more or less know what a floating number is. As a quick reminder, it's a number which is represented as follows: `sign * significand * (base ^ exponent)`

* _Sign_ here can be either -1 or 1. 
* _Significand_ \(also called _mantissa_\) always starts with '**1.**' \(note the dot at the end!\). Since it always starts with 1, there is no need to store that 1 in actual bit representation when we get to it.
* _Base_ is 2 in our case. 
* _Exponent_ is a scaling factor, it is the number of positions we have to move the dot "." to the right or to the left to get back to our number.

For example binary 101.1 can be represented as `1 * 1.011 * (base ^ 2)`, sign is 1, significand is 1.011 and we need to scale it two positions to the right, so exponent is +2. 

In order to convert a real decimal number into bits of `double` we can do the following steps:

1. Get full binary representation of the decimal number we are trying to convert. It will most likely contain the repeating fractional part unless it can be represented as `P / Q` where `Q` is an exact power of 2 and `P` is an integer. For example 3/8 would terminate in binary representation, 1/5 will not.
2. Take 53 bits starting from the first 1. This is because IEEE 754 double has 53 bits of precision. You can look up those number in corresponding Wikipedia table.
3. Figure out what the exponent should be. In our case it will be -4. **TODO**

### Literals conversion

So, how C literals are parsed into doubles? Apparently, according to this StackOverflow [answer](https://stackoverflow.com/a/649108/211906) and [C99 standard](http://c0x.coding-guidelines.com/6.4.4.2.html), there are no limitations on length of double literals \(at least I don't see it in the grammar and I can use literals with thousands digits and it compiles just fine\). The double representation we'll get should be the closest representable number or one before or after it, depending on the implementation. So yes, you can use literals like 0.123456789012345678901234567890 with 30 digits, but most of those digits would be wasted since it's too precise to be represented in double precision format. 

To quote from C99 standard:

> The significand part is interpreted as a \(decimal or hexadecimal\) rational number; the digit sequence in the exponent part is interpreted as a decimal integer. For decimal floating constants, the exponent indicates the power of 10 by which the significand part is to be scaled. \[...\] For decimal floating constants, and also for hexadecimal floating constants when `FLT_RADIX` is not a power of 2, **the result is either the nearest representable value, or the larger or smaller representable value immediately adjacent to the nearest representable value, chosen in an implementation-defined manner**.

### 







