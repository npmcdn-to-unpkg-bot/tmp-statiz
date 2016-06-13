###Background
*I've been hanging out at the Recurse Center over the last seven weeks. People come to RC for lots of different reasons, from different backgrounds. However, everyone has one thing in common: they want to get **dramatically** better at programming.*

One of my primary goals at RC was to get up to speed on the [Rust](https://rust-lang.org) programming language. Personally I'm a fan learning many programming languages of all shapes and sizes. Programming languages (and technologies in the computing world) come and go at a furious rate. If you decide to adopt the next over-hyped language every-time someone tells you to, you probably won't ever get the chance to get really good at something. However, I do think it's super important to grok something new every now and then to assimilate a new paradigm into your programmer's toolkit (akin to the Borg mentality).

I'm not going to wax poetic about the virtues of using (or learning) Rust. The learning curve is admittedly frustratingly high, arguably higher if you have had prior C or C++ exposure. Furthermore I only know of two companies rumored to be using Rust aside from Mozilla, and at the time of this writing neither of them have deployed it to production yet to the best of my knowledge. Though it will probably never dethrone C++ I still think it is worth learning (at least for me). Someone more well-versed than me can probably explain in detail the cool reasons why you should learn Rust offers. But hey, this is something that interests me and I'm going to keep at it.

Taking the time to learn Rust has been very fruitful during my time at RC. It simply would not have been possible for me to learn at this pace if I was working concurrently in industry. You learn about a lot of things along the way from all the incredibly detailed messages the borrow checker (compiler) throws at you when you try to return pointers for objects created during the lifetime of a function (turns out this is dangerous for a non-garbage collected language as you cannot reasonably predict how long the reference is valid for) to accessing shared (unprotected) mutable state across multiple threads (convince yourself that this is a terrible idea) and everything in-between. While such things seem obvious on paper, these are still incredibly common mistakes but Rust makes it impossible for these things to happen (or at least makes it incredibly difficult for you to do).

###Ok so how does metaprogramming come in?

As a statically typed systems language, Rust requires you to specify the length of your arrays at compile time, like C++ (even Java and Scala). Arrays are also strictly fixed length and stack-allocated in Rust. In more dynamic languages Arrays are usually a misnomer and are better described as ArrayLists (looking at you JavaScript). You don't really have to worry about declaring length - and you can trivially grow the size of your array in JavaScript during runtime.

```
// declares an array of 15 32-bit unsigned integers initialized to 0
// (on the RHS, arrays can be specified to a default value)
let my_array:[u32; 15] = [0; 15];
```

Arrays have a type signature of **[ *type* ; *length* ]**

```
// you can't do something like this:

fn do_something (len: usize) {
  let my_array:[u32; len] = [0; len];
}

// You end up with a compile time error like this
// error: no type for local variable 26
// src/node.rs:8     let my_array:[u32; len] = [0; len];
```

To note, this post was inspired when I was working on my [Ailmedak](https://github.com/aaliang/ailmedak) project. Ailmedak is my interpretation of a Distributed Hash Table described by the [Kademlia paper](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf). For added fun and challenge it avoids using dynamic memory as much as possible. It requires an extremely reliable and low latent network connecting nodes before a significant performance advantage over heap allocated nodes can be realized (in  other words there's no real advantage of using Ailmedak except possibly in an embedded system).

The following examples I cite will be in context of simplified things that I've done in the Ailmedak codebase.

```
trait Node {

  // Nodes have ids. and right now they're an array of 8-bit unsigned
  // integers (aka byte array of length 20)
  fn generate_new_id () -> [u8; 20];

  // Nodes do stuff with ids of other nodes
  fn do_something_with_id (&self, id: &[u8; 20] );
  fn consume_id (&mut self, id: [u8; 20])

}
```

In the contrived example above we have functions that either return or accept as input ids which are 20-length byte arrays. Suppose that this works great for us at first. However, imagine that sometime in the future our requirements change and we change our definition of an id as a 33-length byte array instead.

If you're a perfectly reasonable person in order to support ease of development there are probably several approaches to use.

The most obvious (and least painful) is to use a [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) which is a growable list type. This is analogous to Python's List, C++'s Vector, Java's ArrayList, and JavaScript's Array. However, for the sake of being particularly masochistic let's say this is not an option. Vecs in Rust are heap allocated (dynamic memory). There is some overhead when allocating heap memory (it's non deterministic in time) whereas stack allocation is relatively trivial from a performance perspective (all you do is increment the stack pointer).

Well, every time you want to change the id length, you'd have to change the type signature
```
trait Node {
  fn generate_new_id () -> [u8; 33];
  fn do_something_with_id (&self, id: &[u8; 33] );
  fn consume_id (&mut self, id: [u8; 33] );
}
```

But then you'd have to do it for every id that gets returned, or used as input and this could get somewhat painful if you have a lot of functions.

You could, however use type aliases:
```
type NodeId = [u8; 33];
trait Node {
  fn generate_new_id () -> NodeId;
  fn do_something_with_id (&self, id: &NodeId);
  fn consume_id (&mut self, id: NodeId);
}
```

This way you only have to change it in one place. However this is still flawed for future use. Suppose in the event that you now want the process to support operations on multiple types of nodes, some with ids of length 5, some length 10, some length 20, etc. The above trait only works for one specific sized array - you'd have to copy it in it's entirety into a separate Node5, Node10 trait etc.

###Enter Generics

Generics in Rust behave similarly to generics in Java/Scala as well as templates in C++. They provide a facility to provide a type as a parameter to structs, impls, traits, enums, and functions.

```
trait Node <T> {
  fn generate_new_id () -> T;
  fn do_something_with_id (&self, id: &T);
  fn consume_id (&mut self, id: &T);
}
```

and then you could provide a concrete type
```
type NodeId = [u8; 20];
struct MyNode;
impl Node <NodeId> for MyNode {
  fn generate_new_id () -> NodeId;
  fn do_something_with_id (&self, id: &NodeId);
  fn consume_id (&mut self, id: &NodeId);
}
```
This would work pretty well... until the code evolves even more. Suppose you want to provide default implementation for generate\_new\_id in the Node trait. (similar to virtual functions in C++)

A non-generic implementation of a generate\_new\_id could possibly look like this:
```
  fn generate_new_id () -> [u8; 20] {
    // using unsafe here to avoid specifying a default
    // we're overwriting the region of memory anyways
    // could easily default to 0 for this non-generic version
    let mut id_buf:[u8; 20] = unsafe { mem::uninitialized() };
    let mut nrg = rand::thread_rng();
    // iterator to generate a random byte value
    let iter = nrg.gen_iter::<u8>();
    for (x, i) in iter.take(20).enumerate() {
      id_buf[x] = i;
    }
  }
```

When we try to place this into our generic trait we end up losing too much granularity, annotated in the code
```
trait Node <T> {

  fn generate_new_id () -> T {
    let mut id_buf:T = unsafe { mem::uninitialized() };
    let mut nrg = thread_rng();
    // looks like we have to hardcode the type here but to what?
    let iter = nrg.gen_iter::<???>();
    // not even sure how much to take with the generic T
    for (x, i) in iter.take(???).enumerate() {
      id_buf[x] = i;
    }
  }

  fn do_something_with_id (&self, id: &T);

  fn consume_id (&mut self, id: &T);
}
```
With the ??? marked sections above, there's no way to figure out:
1. The length of the id
2. The type of the Array. There's no reason why the array can't be an array of u32's or f64's etc.

We have to go back to the [u8; 20] kind of way, except the length (and type) needs to be parametized. We need something better.

###Behold, Macros
Macros are not a concept unique to Rust. They've been around in languages such as C, and Common Lisp. There's a lot of crazy shit you can do with macros.

Macros in Rust look a lot like functions, except they emit code. They even behave sort of similarly when you expand them except they are expanded prior to the compilation phase. They can be expanded recursively and even have their own version of pattern matching to boot. A restriction is that they (the values they operate on) must be resolvable at runtime. The way to think about macros in Rust (or even in C) is that they *'return'* code in-place that behaves as if you had written it there yourself.

The syntax for invoking a macro is relatively flexible.
generally goes like:
```
macro_rules! /* NAME OF YOUR MACRO */ {
  (/* MACRO ARGUMENTS (DOES NOT HAVE TO BE RUST CODE)*/) => {
    /* WHATEVER YOUR MACRO SHOULD EXPAND TO (RUST CODE) */
  }
}
```
With that we can have a macro expanding into a trait for our Node:
```
macro_rules! meta_node {
  // macro is invoked with an ident (called $name)
  // followed by "(id_len = "
  // followed by an expr (called $length)
  ($name:ident (id_len = $length:expr)) => {
    pub trait $name <T> where T: PartialEq + PartialOrd + Rand {
      fn do_something_with_id (&self, id: &T) -> &[T; $length];
      fn consume_id (&mut self, id: T) -> [T; $length];
      fn generate_new_id () -> [T; $length] {
        let mut id_buf:[T; $length] = unsafe { mem::uninitialized() };
        let mut nrg = thread_rng();
        let iter = nrg.gen_iter::<T>();
        for (x, i) in iter.take($length).enumerate() {
          id_buf[x] = i;
        }
        id_buf
      }
    }
  }
}
```
If we invoke our macro like so:
```
const ID_SIZE:usize = 20;
meta_node!(ASizedNode (id_len = ID_SIZE));
type NodeId = [u8; ID_SIZE];
```
We are essentially taking everything to the right of the =>, replacing all instances of $name with ASizedNode, all instances of $length with 20, and then splatting the result as code. You end up with a trait called ASizedNode generating (and operating on) 20-length byte arrays Pretty powerful huh?!

Finally you can create a concrete implementation of ASizedNode
```
pub struct KademliaNode;

impl ASizedNode<u8> for KademliaNode {
  fn do_something_with_id (&self, id: &NodeId) -> {
    println!("id: {:?}", id);
  }
  fn consume_id (&mut self, id: NodeId) -> {
    //do nothing else
  }
}
```

And that's pretty much it! Yeah, if we only cared about ids as inputs to functions  (and didn't care as much when we return) there would be a way to do this using array slices (slices don't require you to annotate the lengths) as they're really just borrowed sections of Arrays (or Vecs) with a start and and end. But then you lose some type safety without the lengths baked into the type signature. Please note that this only counts as compile-time metaprogramming. Rust does not have a reflection API that I'm aware of yet (and it probably wouldn't be an ideal solution as Rust emphasizes zero-cost abstractions :) )
