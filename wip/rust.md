# An Introduction To Rust

Rust is an interesting problem.

It's a programming languages with a lot of issues. Two of them are what I believe are the biggest.

1. The Rust committee is made up of morons.
1. The language is hard, and the docs are terrible in fixing it.

My goal with this article, much like the Go article, is to introduce Rust to the reader from the context of someone
already familiar with web development. If you are not already rather familiar with HTTP and it's patterns, start with
another language like Java or C#.

______________________________________________________________________

Unlike go, we need to start with some language basics.

## The Borrow Checker

The obvious first thing to talk about is the borrow checker. Memory safety is a major part of why the language is
"semi"-popular nowadays, but it can be hard to wrap one's head around. Here's a quick example of something you **can't**
do.

```rust
fn print(s: String) {
    println!("{}", s);
}

fn main() {
    let a: String = "hi".to_string();
    print(a);
    print(a); // a doesn't exist anymore
}
```

The reason is that variables can only have one owner. Giving it to the function means that the function now owns that
variable, and when the function leaves scope, the variable is deleted.

To fix it you take a reference of the string and pass that instead.

```rust
fn print(s: &String) {
    println!("{}", s);
}

fn main() {
    let a: String = "hi".to_string();
    print(&a);
    print(&a); // a doesn't exist anymore
}
```

What the docs will say is that taking and passing a reference is how to borrow variables. This I think is where the docs
need to be more clear. It's not that a reference is how you borrow things. Lifetimes are how you borrow things, and you
can only apply a lifetime to a pointer. Here's a filled out version of the previous function with the lifetime intact.

```rust
fn print<'a>(s: &'a String) {
    println!("{}", s);
}
```

What this means is that a lifetime is passed into the function by the caller, and that lifetime is tied to the input
variable. Simple situations like this can elide the lifetimes making them optional, though still there.

What this means is simple, you can only have one **hard** instance of the variable, but can pass around references as
long as the original ownership is still in scope. There is one situation where this isn't the case. If you need multiple
primary references to a thing, you can use an `Rc`. `Rc` is a reference counting container. You close the container, and
it ticks up a number, when the container goes out of scope it ticks down. Once all references are gone the container is
properly deleted.

I've not used the regular `Rc`. `Arc` is much more common in web dev, as it's a thread safe reference counter used for
shared state like database connection pools.

There's some additional nuance when it comes to instance methods, mostly because they don't actually exist, but that's
next.

## Data Structures

Rust, similar to most languages that use the C FFI, is not object oriented. It does not have classes, it has `struct`s.
Here is an example in C of how you can simulate instance methods.

```c
#include <stdio.h>

struct example {
    char *data;
};

struct example example_new(char *data) {
    struct example ret;
    ret.data = data;
    
    return ret;
}
void example_print(struct example *self) { printf("hi %s\n", self->data); }

int main() {
    struct example hi = example_new("dave");

    example_print(&hi);
}
```

Both go and rust use a similar style, though they both better simulate instance methods.

```rust
struct TestThing {
    data: String,
}

impl TestThing {
    fn new(s: impl ToString) -> Self {
        Self {
            data: s.to_string(),
        }
    }
    
    fn print(&self) {
        println!("{}", self.data);
    }
}
```

Rust has similar constraints to C in that you cannot overload functions, and since it doesn't have objects, uses static
methods to instantiate an object. The convention is to name the method `new`. `impl`allows you to implement functions
for a struct. In the context of an impl there are two important notes. The first is that`Self`is shorthand for what the
impl type is. The second is that if you use`self`or`&self` as the first parameter, you can then call that function as if
it were an instance method.

```rust
let t = TestThing::new("dave");
t.print();
```

Something else you can note is that if the last expression is a return, you can omit both the semicolon and the return
keyword.

The important part of the `print` function is that it accepts self as a reference. This is important because what you
are actually calling is this.

```rust
let t = TestThing::new("dave");
TestThing::print(&t);
```

This means that if `self` isn't a pointer, it will consume the object. It can be confusing that an instance method could
consume the object, but that's because it's not an instance method, those don't exist in Rust.

## Traits

Traits are kind of like interfaces in reverse. With an interface, you define the interface first, and then you implement
that interface with the object. The interface is kind of like a blueprint for the object.

One problem with interfaces is that you can have multiple implemented by the same object. If you have multiple
interfaces that define the same method (or even method name), you now have a problem. As stated before, Rust doesn't
support overloading.

Instead the solution is traits. A trait by itself is effectively an interface. You can define the trait, with methods.
Here's an example of a basic trait (one that's relatively useless in the real world).

```rust
trait StringLike {
    fn stringify(&self) -> String;
}
```

You then can create an implementation for your trait.

```rust
impl StringLike for TestThing {
    fn stringify(&self) -> String {
        self.data.clone()
    }
}
```

Note that I'm cloning this here, that's because another implementation basically makes it required. That implementation
is more generic. One piece of magic in rust is that you can implement a trait based off of a trait. Here's an example.

```rust
impl<T: ToString> StringLike for T {
    fn stringify(&self) -> String {
        self.to_string()
    }
}
```

This is very common. `ToString` itself is rarely implemented. Most of the time you implement `Display`, for which there
is a blanket `ToString` implementation.
