# optimize_common_cases
Ways to optimize for the common case.

## Background

In code, sometimes we have functions for which we need to support all
sorts of inputs but there may be common cases for which we would like
to optimize.  For example, say you are writing a jpeg image processor:

```c++
void process_image(int width, int height, char *data);
```

It might be that most of your images are 4000 x 3000 pixels.  What
you'd like is to specialize for that case and do some optimizing.

## Methods

### Write an extra function

You could write a special version of the code to handle the case of
4000x3000 like this:

```c++
void process_image_helper(int width, int height, char *data);
void process_image_helper_4000x3000(char *data);

void process_image(int width, int height, char *data) {
  if (width == 4000 && height == 3000) {
    process_image_helper_4000x3000(data); // *** specialized
  } else {
    process_image_helper(width, height, data); // *** regular
  }
}
```

The problem with this technique is that you need to write your
function twice, once for the general case and once for 4000x3000.
This might lead to a lot of code duplication.

If you do it this way, you want to make that `if` statement as deep as
possible in the function.  For example, if there are a few steps to
image processing and only some of them benefit from the optimization
then put the branch around just that code, that way less code is
duplicated.

```c++
void process_image_headers(int width, int height, char *data);
void process_image_contents(int width, int height, char *data);
void process_image_contents_4000x3000(char *data);

void process_image(int width, int height, char *data) {
  // No optimizations in the headers.
  process_image_headers(width, height, data); // *** common

  if (width == 4000 && height == 3000) {
    process_image_contents_4000x3000(data); // *** specialized
  } else {
    process_image_contents(width, height, data); // *** regular
  }
}
```

This will minimize the amount that is duplicated.  You'll probably
still have some duplication in the places where it is too difficult or
tedious to tease apart the optimization from the duplicated code,
however.  For example:

```c++
void process_image_contents(int width, int height, char *data) {
  if (width == 4000 && height == 3000) { // *** branch early
    // *** specialized
    for (int i = 0; i < 3000; i++) {
      // do some row preparations.
      for (int j = 0; j < 4000; j++) {
        // do some height preparations.
        // do the pixel processing.
      }
    }
  } else {
    // *** regular
    for (int i = 0; i < height; i++) {
      // do some row preparations.
      for (int j = 0; j < width; j++) {
        // do some height preparations.
        // do the pixel processing.
      }
    }
  }
}
```

In this case, we have duplicated the `for` loop.  But what if only the
pixel processing can take advantage of the known sizes?  We could
write this:

```c++
void process_image_contents(int width, int height, char *data) {
  for (int i = 0; i < 3000; i++) {
    // do some row preparations.
    for (int j = 0; j < 4000; j++) {
      // do some height preparations.
      if (width == 4000 && height == 3000) { // *** branch later
        // do the *optimized* pixel processing.
      } else {
        // do the regular pixel processing.
      }
    }
  }
}
```

Now we have less code duplication, which is good.  But the `if` branch
is getting run at each iteration, which will be slow!

### Templating

Templating doesn't really help us here.  We could write this:

```c++
void process_image_helper(int width, int height, char *data);

template <int width, int height>
void process_image_helper(char *data);
```

And now we can write our code like this:

```c++
void process_image(int width, int height, char *data) {
  if (width == 4000 && height == 3000) {
    process_image_helper<4000, 3000>(data); // *** specialized
  } else {
    process_image_helper(width, height, data); // *** regular
  }
}
```

The template perhaps looks a little cleaner but behind the scenes the
same thing is happening.  The compiler will "mangle" the name of the
templated function to look something like this:

```
_Z20process_image_helperILi4000ELi3000EEvv
```

Just as we made a function with the name `4000x3000`, the compiler
made a function with the name `ILi4000ELi3000EEvv`.

### Optimizer

Perhaps the cleanest way that we can do this is by counting on the
optimizer to do it for us:

```c++
void process_image(int width, int height, char *data) {
  if (width == 4000 && height == 3000) {
    // Allow compiler to make an optimized version.
    process_image_helper(width, height, data);
  } else {
    process_image_helper(width, height, data);
  }
}
```

In both branches we have same code!  We might expect the compiler to
optimize away the `if` branch but it seems that the compiler will
optimize the function inside instead.  Here's some example code for
computing a sum of consecutive integers:

```c++
#include <cmath>
#include <stdio.h>

// This is a double so that the compiler
// will not optimize away the loop.
int getSum(double val) {
  int sum = 0;
  for (int i = val; i < val + 1000; i++) {
    sum += i;
  }
  return sum;
}

int main(int argc, char *argv[]) {
  if (argc == 5) {
    printf("%d\n", getSum(argc));
  } else {
    printf("%d\n", getSum(argc));
  }
  return 0;
}
```

In the assembly code, we get this:

```asm
main:                                   # @main
        push    rax
        mov     esi, 504500
        cmp     edi, 5
        je      .LBB1_13
```

We can see a comparison against `5` and, if that comparison is true,
we'll use the number `504500`, which is the correct result.  The
optimizer has completely removed the `for` loop.

The nice thing about this technique is that we can add more cases
quite easily.

```c++
int main(int argc, char *argv[]) {
  if (argc == 5) {
    printf("%d\n", getSum(argc));
  } else if (argc == 6) {
    printf("%d\n", getSum(argc));
  } else {
    printf("%d\n", getSum(argc));
  }
  return 0;
}
```

becomes:

```asm
main:                                   # @main
        push    rax
        cmp     edi, 6
        je      .LBB1_1
        cmp     edi, 5
        jne     .LBB1_4
        mov     esi, 504500
        jmp     .LBB1_16
.LBB1_1:
        mov     esi, 505500
        jmp     .LBB1_16
```

The issue is that it will only work if `getSum` is in the same
translation unit.  Either the function is declared in the same file or
it is in a header that is included by this file.

```c++
#include <cmath>
#include <stdio.h>

int getSum(double val);

int main(int argc, char *argv[]) {
  if (argc == 5) {
    printf("%d\n", getSum(argc));
  } else if (argc == 6) {
    printf("%d\n", getSum(argc));
  } else {
    printf("%d\n", getSum(argc));
  }
  return 0;
}
```

leads to `getSum` being called for all cases:

```c++
main:                                   # @main
        push    rax
        cmp     edi, 5
        je      .LBB0_1
        cmp     edi, 6
        jne     .LBB0_4
        movsd   xmm0, qword ptr [rip + .LCPI0_1] # xmm0 = mem[0],zero
        jmp     .LBB0_5
.LBB0_1:
        movsd   xmm0, qword ptr [rip + .LCPI0_0] # xmm0 = mem[0],zero
        jmp     .LBB0_5
.LBB0_4:
        cvtsi2sd        xmm0, edi
.LBB0_5:
        call    getSum(double)@PLT
```

To prevent the caller from accidentally missing out on the optimization, we
should declare `getSum` with a helper like this:

```c++
#include <cmath>
#include <stdio.h>

// This is a double so that the compiler
// will not optimize away the loop.
static int getSumHelper(double val) {
  int sum = 0;
  for (int i = val; i < val + 1000; i++) {
    sum += i;
  }
  return sum;
}

int getSum(int val) {
  if (val == 5) {
    printf("%d\n", getSumHelper(val));
  } else if (val == 6) {
    printf("%d\n", getSumHelper(val));
  } else {
    printf("%d\n", getSumHelper(val));
  }
  return 0;
}
```

And now other files can call `getSum` and get the optimized version.
