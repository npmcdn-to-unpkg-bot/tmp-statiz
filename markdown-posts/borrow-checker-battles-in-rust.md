kely the first thing that baby rustaceans will bang their heads on is the concept of lifetimes and the borrow checker. One of the things I took for granted in virtually every other programming language that I used was the ability to call methods off of an object without too much fuss. Calling methods needs to be handled drastically differently in rust-lang.

In the listing below I'm trying to model a simple type representing a node in a simple neural network:

* An instance of a Neuron will contain a 32 bit integer for the number of inputs it takes *num_inputs*
* It will also contain a vector of weights (as 64-bit floating points) that dictate the value of those inputs *weights*

The corresponding struct probably looks something like this:
```
struct Neuron {
  num_inputs: i32,
  weights: Vec<f64>
}
```

We'll also want to create the rust analogue to a static method *(Neuron::new)* which serves as a constructor by randomly generating a total of *ninputs* floating points to populate a Neuron struct accordingly. *Note that in the listing we're actually creating ninputs+1*
```
impl Neuron {
  fn new (ninputs: i32) -> Neuron {
    let mut weight_vec = Vec::new();
    // add {ninputs+1} weights...
    for _ in 0..(ninputs+1) {
      weight_vec.push(rand::random::<f64>());
    }
    Neuron{num_inputs: ninputs, weights: weight_vec}
  }
}
```



Now let's say we want to do some quality logging at some point and we would like to print out each of the weights in a neuron's vector. For convenience let's make it a method! Hmm yeah...! let's add it to `impl Neuron` shall we?

```
impl Neuron {

      /* ... blah blah... fn new still looks the same */

          fn print_weights (self) {
                    for weight in self.weights {
                                  println!("{}", weight);
                                          }
                                              }
}
```

Now we can instantiate an instance, and then print the weights like so:
```
fn main() {
      let a_neuron = Neuron::new(2);
          a_neuron.print_weights();
}
```
With any luck we'd probably see something like this:
```
0.7330616595228169
0.6183962989161881
0.79081893581957
```

But what if we wanted to print it again?
```
let a_neuron = Neuron::new(2);
a_neuron.print_weights();
a_neuron.print_weights();
```
This is where everything falls apart and you start to turn to StackOverflow (by way of google of course). You'll most likely see a compiler error along the lines of:
```
use of moved value `a_neuron`
```

So what's going on here? If we take a look at the signature for `Neuron::print_weights`:

```
fn print_weights (self)
```

It turns out that because of the`self` parameter, when the method is called  `a_neuron.print_weights` implicitly takes ownership of `self` (these are move semantics). When `a_neuron.print_weights` is called the second time, ownership is never yielded back to the main function (or whatever function we calling it from).

In order to make this work, we need to **declare print_weights as a borrow, as well as the access to self.weights**:

```
fn print_weights (&self) {
  for weight in &self.weights {
    println!("{}", weight);
  }
 }
```

Our full listing ends up looking something like this:
```
extern crate rand;

struct Neuron {
  num_inputs: i32,
  weights: Vec<f64>
}

impl Neuron {
  fn new (ninputs: i32) -> Neuron {
    let mut weight_vec = Vec::new();
    //add {ninputs + 1} weights
    for _ in 0..(ninputs+1) {
      weight_vec.push(rand::random::<f64>());
    }
    Neuron{num_inputs: ninputs, weights: weight_vec}
  }

  fn print_weights (&self) {
    for weight in &self.weights {
      println!("{}", weight);
    }
  }
}

fn main() {
  let a_neuron = Neuron::new(2);
  a_neuron.print_weights();
}
```

And hopefully we see output along the lines of:
```
0.45597887767522716
0.6523807976419703
0.43939320025023854
0.1353881594997136
0.45597887767522716
0.6523807976419703
```

Please note that I do not claim to know everything and all there is to know about Rust! If I got anything wrong and if there is anything that I should mention, please be sure to let me know!
