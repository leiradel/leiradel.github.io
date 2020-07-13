---
layout: post
title: Checking Platform With Macros
---

Recently I had some issues with platform dependent code not being compiled when it should, or being compiled when it shouldn't. It turned out that the way I was checking for the current platform has some opportunities for bug because of the way the pre-processor deals with misspelled macro names.

After some investigation, I've changed the code to use a more robust way to test the current platform that I's like to share.

> This post is **not** about finding what the current build platform is, but how to **test** for it.

## `#ifdef`

This is probably the most common way to deal with different targets in code. Just define a macro that represents the current platform and test for it being defined. When chaining many tests, the `defined` operator can be used to avoid nesting.

```c
#ifdef MY_PLATFORM_A
// Specific code for platform A
#endif

#if defined MY_PLATFORM_B
// Specific code for platform B
#elif defined MY_PLATFORM_C
// Specific code for platform C
#endif
```

Simple enough and easy to understand, but what happens if the macro is misspelled? The build may fail, and that would be the best scenario, but sometimes the code will behave erratically and it can be hard to find the root cause.

Also, it is hard to detect typos in macros to force the build to crash.

## `#if MY_TARGET_PLATFORM == MY_PLATFORM_X`

This solution needs that all `MY_PLATFORM_X` macros are defined to unique integers, and that `MY_TARGET_PLATFORM` is defined to the current target platform.

```c
#define MY_PLATFORM_A 1
#define MY_PLATFORM_B 2
// And so forth

// Test for the current target platform
#if SOME_MACRO_THAT_IDENTIFIES_PLATFORM_A
#define MY_TARGET_PLATFORM MY_PLATFORM_A
#endif
```

At first this may look like a better alternative to `#ifdef`. However, it's not really an improvement, and it hides additional opportunities for bugs.

The issue is, if the developer misspells the macro names the test can end up being `true` or `false` even though it isn't supposed to, just like in the previous solution.

The reason is that the pre-processor will default undefined macros in `#if` tests to zero instead of issuing an error:

```c
// The following test will be #if 0 == 0
#if MY_MISSPELLED_TARGET_PLATFORM == MY_MISSPELLED_PLATFORM_A
// This will always be compiled
#endif

// The following test will be #if 0 == MY_PLATFORM_B
#if MY_MISSPELLED_TARGET_PLATFORM == MY_PLATFORM_B
// This may or may not be compiled
#endif

// The following test will be #if MY_TARGET_PLATFORM == 0
#if MY_TARGET_PLATFORM == MY_MISSPELLED_PLATFORM_C
// This may or may not be compiled
#endif
```

Sometimes the bug will be easy to fix, but sometimes a lot of time will be wasted trying to find the real cause. This is also hard to check for typos during the build.

Some bugs can be avoided by making sure none of `MY_PLATFORM_X` macros is zero, but there is a better solution.

## `#if MY_PLATFORM_IS(x)`

```c
#if MY_PLATFORM_IS(MY_PLATFORM_A)
// Specific code for platform A
#elif MY_PLATFORM_IS(MY_PLATFORM_B)
// Specific code for platform B
#endif
```

At first this looks like just the previous solution under a new dress, but the way the pre-processor works makes this a good solution depending on how `MY_PLATFORM_IS` is defined.

A standard way to define this macro is:

```c
#define MY_PLATFORM_IS(x) (MY_TARGET_PLATFORM == (x))
```

With that definition, if `MY_PLATFORM_IS` is misspelled the pre-processor will stop with an error:

```c
#if MY_MISSPELLED_PLATFORM_IS(MY_PLATFORM_A)
// Some code
#endif
```

```
$ gcc -c test.c
test.c:1:19: error: missing binary operator before token "("
 #if MY_MISSPELLED_PLATFORM_IS(MY_PLATFORM_A)
                   ^
```

It's still possible however to have bugs when `x` is misspelled, but there's a way to define `MY_PLATFORM_IS` such that it will also fail in that case:

```c
#define MY_PLATFORM_IS(x) ((1 / ((x) != 0)) && (MY_TARGET_PLATFORM == (x)))
```

When `MY_PLATFORM_IS` is misspelled, the pre-processor will fail like before, but now it will also issue an error if `x` is misspelled:

```c
#define MY_PLATFORM_IS(x) ((1 / ((x) != 0)) && (MY_TARGET_PLATFORM == (x)))

// This will be evaluated to ((1 / (0 != 0)) && (MY_TARGET_PLATFORM == 0)
#if MY_PLATFORM_IS(MY_MISSPELLED_PLATFORM_A)
// Some code
#endif
```

Since `0 != 0` is zero, the pre-processor will stop with a division by zero:

```
$ gcc -c test2.c
test2.c:1:38: error: division by zero in #if
 #define MY_PLATFORM_IS(x) ((1 / ((x) != 0)) && (MY_TARGET_PLATFORM == (x)))
                                      ^
test2.c:4:5: note: in expansion of macro ‘MY_PLATFORM_IS’
 #if MY_PLATFORM_IS(MY_MISSPELLED_PLATFORM_A)
     ^~~~~~~~~~~~~~
```

## Failing the Build

The last touch is forcing developers to use the `MY_PLATFORM_IS` macro instead of testing for equality. Thankfully, it's easy to check this with `grep`, which can be added as a build step in any build system out there in one way or another.

```
all: checkplat

checkplat: FORCE
	@grep --exclude platform.h -RIw MY_PLATFORM_IS \
		| grep == | wc -l \
		| xargs expr 1 - > /dev/null

FORCE:
```
