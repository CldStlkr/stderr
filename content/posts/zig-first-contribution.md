+++
title = "My First Contribution to Zig: Implementing insque() and remque()"
draft = false
tags = ["zig", "open-source", "systems-programming"]
ShowReadingTime = true
date = 2026-02-12
+++

I recently made my first contribution to the Zig programming language by implementing
`insque()` and `remque()`, two doubly-linked list functions from the C standard
library.

## What are insque and remque?

These functions are used to insert and remove from queues built from doubly-linked
lists, with the function signatures below:

```c
void insque(void *element, void *pred);
void remque(void *element);
```

Their implementations were surprisingly simple, just involving casting pointers
into `node`\
structs and hooking up the new node to its neighbors or the opposite if removing,
which is why I chose this to be my first contribution.

## The Journey

### Initial Setup

Setup was fairly easy, though there were some hiccups after finishing the implementation
that I will elaborate on later

All I needed was to fork the Zig project from Codeberg and look at the Zig implementations
of other headers like `string.zig` to get an idea of what to change.

### The Implementation

Like I said before, the implementation in C was so simple I almost couldn't
believe it. This may be in large part to having written an RTOS kernel in C,
of which a central component is the intrusive doubly-linked
lists that are embedded into my Task Control Blocks.

All this to say, that I still had a somewhat difficult time translating my the C
code into idiomatic Zig code. Specifically, the `insque()` logic.

The below code is the entirely of the `insque()` musl implementation:

```c
#include <search.h>

struct node {
  struct node *next;
  struct node *prev;
};

void insque(void *element, void *pred) {
  struct node *e = element;
  struct node *p = pred;

  if (!p) {
    e->next = e->prev = 0;
    return;
  }
  e->next = p->next;
  e->prev = p;
  p->next = e;
  if (e->next)
    e->next->prev = e;
}
```

Its extremely straightforward, with a node definition and `void *` as the type
for the function args `element` and `pred`, which is short for predecessor.

From there, the two void pointers are cast to node pointers, and then there
is a check for if `p` (the node-casted value of `pred`) is null, in which case
we null the next and previous node pointers for `e`, since if it has no
predecessor, it must be the the only value in the list.

Once null `p` is accounted for, we can simply hook up the pointers accordingly,
accounting for if `p` was not the last node, and would therefore have a
next node that would now need to be the next node of `e` instead

In contrast, here is my full implementation of the Zig equivalent:

```zig
const std = @import("std");
const builtin = @import("builtin");
const symbol = @import("../c.zig").symbol;

comptime {
  if (builtin.target.isMuslLibC() or builtin.target.isWasiLibC()) {
    symbol(&insque, "insque");
    symbol(&remque, "remque");
  }
}

const Node = extern struct {
  next: ?*Node,
  prev: ?*Node,
};

fn insque(element: *anyopaque, pred: ?*anyopaque) callconv(.c) void {
  const e: *Node = @ptrCast(@alignCast(element));

  if (pred) |p_ptr| {
    const p: *Node = @ptrCast(@alignCast(p_ptr));
    e.next = p.next;
    e.prev = p;
    p.next = e;

    if (e.next) |next| {
      next.prev = e;
    }
  } else {
      e.next = null;
      e.prev = null;
    }
}
```

First, you can see that I needed to tell the Zig compiler when building for musl
or wasi, export these two functions (with their original C names) so that they can
be callable from C code. The symbol() function handles all the @export boilerplate
that allows them to be a part of the public C API.

```zig
if (builtin.target.isMuslLibC() or builtin.target.isWasiLibC()) {
  symbol(&insque, "insque");
  symbol(&remque, "remque");
}
```

For similar reasons, I also needed to create the `Node` struct as extern, so that
it is also compatible with the C ABI (Different than API! ABI defines how data
is laid out in memory and how functions are called, while API is just the
interface you code with/against (function signatures, etc)).

```zig
const Node = extern struct {
  next: ?*Node,
  prev: ?*Node,
};
```

`*anyopaque` is Zig's equivalent to C's `void *`; It's an opaque pointer to any
type. Zig, similar to Rust, supports Optionals through the use of the `?`
syntax. We use it for the `pred` pointer because it is possible for
that to be validly null. the `callconv(.c)` as the return value ensures we
use the C ABI, which makes it compatible with C code.

```zig
fn insque(element: *anyopaque, pred: ?*anyopaque) callconv(.c) void
```

Since we receive generic pointers, we need to cast them to the concrete `*Node`
type so that we can access the `next` and `prev` fields. `@ptrCast` changes the
pointer type, while `@alignCast` tells Zig to trust that the pointer has
the correct alignment for a `Node` struct.

Understanding those intricacies was ~80% of the difficulty, and the other 20%
came from my unfamiliarity with how to unwrap optional values in Zig.
In Rust, there are many ways to unwrap an optional, with the simplest being
`.unwrap()`. The analog for this in Zig is `.?`, but this would panic upon
reaching a null value in both languages. I had to look up the documentation for
how I could do the equivalent of `if let Some(ptr) = ...` in Zig, which turned
out to be `if (p) |p_ptr| {...}`

### Build Issues

I don't want to bog you down with my trials and tribulations of figuring out how
to compile and test the entire project so I will try to keep it short.

In summary I could not get my version
of Zig (Some dev build of 0.16.x) to compile and test my changes, so I had to
build using C and cmake which took a pretty long time even with make -j$(nproc)
using all 12 cores of my CPU.

Then, after ensuring my changes passed the test suite,
I submitted my PR and within 15 minutes got 3 different comments on what I needed
to change in my code, which I spent a bit trying to fix and then realizing that
I had merge conflicts and was adding unnecessary changes leading me down a rabbit
hole of making 4+ commits to my PR changing 1-10 lines at a time. Not a good look.

## Lessons Learned

- How Zig's build system works
- The confidence to navigate professional Zig and C code
- Building a programming language from source takes a very long time

## Future Contribution Plans

I plan to contribute more, probably staying with the same issue, rewriting C functions
into Zig, since it gives me experience with reading professional C code and
mapping it to
idiomatic Zig. Eventually, I hope to branch out to more complicated stuff, but
for now, I'll
stick to small-scope issues like this until I can understand the black magic
syntax that is used
everywhere else in the Zig code base

## Links

- [My PR](https://codeberg.org/ziglang/zig/pulls/31185)
- [Issue #30978](https://codeberg.org/ziglang/zig/issues/30978)
