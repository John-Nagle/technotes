# On the potential for statically checked backlinks in Rust

(Work in progress)

September, 2025

John Nagle

## Introduction
(MORE)

## The problem
* The C/C++ crowd finds Rust's safety restrictions in this area too strict.

## The usual workarounds

### Putting all items in an array and using array indices as if they were pointers.
* Can have the equivalent of dangling pointers
* Can have orphaned items
* Most of the problems of raw pointers, although subscript checks prevent memory corruption.

### Unsafe code
* All the usual problems.
* If you have data structures complex enough to need this, the chance of programmer error is high.

### Rc/Weak/RefCell
* Works fine
* Some run time overhead
* **.borrow()** panics if double-borrowed. How does the programmer determine that won't happen?
* Code is easiest to understand if 
* Somewhat verbose

## General concept of the proposal
* Define a type that works like **RefCell**, but where **.borrow()** and **.borrow_mut()** are checked at compile time.
For the purpose of this discussion, we will call that new type, with no run-time checking, **CompiledRefCell**. 
That name is just for discussion purposes; don't waste effort bikshedding on this.
* Define a set of checking rules that are not too difficult to check at compile time but are powerful enough to handle useful complex cases.

The big question: is there a large enough intersection between what we can check easily and what is useful?

### Checking rules
This is the hard part. Borrows must be disjoint. They can be
* Disjoint as to type - RefCell&lt;A&gt; and RefCell&lt;B&gt;, where A and B cannot be the same type, are disjoint.
* Disjoint as to scope - if all **.borrow()** and **.borrow_mut()** calls return results with statically defined scopes, and no two scopes of the same type overlap, the borrows are disjoint.
* Disjoint as to instance - if two items are both borrowed in overlapping scopes, but the borrowed objects can be shown to be different objects, the borrows are disjoint. This is the hard one. If such disjointness analysis is confined to single functions, the checker's job appears manageable.

## The two main patterns
### Single ownership with back references that never outlive the owning reference
(NEED PICTURE)

This is the most common case - A owns B, and B needs to be able to find A. 
A has a unique owner. This pattern covers trees with backlinks, as well as doubly-linked lists. 
It also covers C++ type object inheritance, where the connection from child to parent is maintained automatically.
It can be expressed in Rust with **Rc**, **Weak**, and **RefCell**, but with more dynamism, and less compile time checking, than is appropriate.
When a backlink is needed, Rust's simple compile-time checked ownership model has to be abandoned. That's a high price to pay.

What we really want is single ownership with weak backlinks, combined with a guarantee that no weak backlink can outlive the owning reference, and
that no weak link to a struct is temporarily a strong link at the moment the struct is dropped. That can be checked at run time. Could it be checked
at compile time?

The key here is the use of .upgrade() of weak pointers in a scoped way only.
The same kind of disjointness rules mentioned for RefCell have to be applied to .upgrade() calls.

Working on this for Rust: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=4216d6b42837ff021400b582593a23f4

(MORE)
(WORK IN PROGRESS)
Define

    type OwnerCell<T> = Rc<RefCell<T>>;

This captures the semantics of what's proposed here.
It's an easy way to think about this. 
It does not enforce the single-owner requirement.
Restrictions needed are:

* **OwnerCell** is not cloneable. This enforces the single-owner requirement.

* Each **.borrow()** and **.borrow_mut()** requires its own **downgrade()** call.
The strong pointer created by a downgrade is never used other than for an immediate **.borrow()** and **.borrow_mut()**.
This can be achieved by writing an OwnerCell type that does not expose **upgrade()**.
So **.borrow()** and **.borrow_mut()** would implicitly do the **upgrade()** and panic on failure.

* The scopes of **.borrow()** and **.borrow_mut()** must be disjoint, as above, to prevent such panics. 
That's what we would like to check at compile time.

* When an **OwnerCell** is dropped, it is an error if any borrows are active.
This, too, should be checked at compile time. That's not too hard if all borrows have limited scope.

This can be written in Rust with run time enforcement.
Can these restrictions be checked at compile time?

(NEEDS WORKED EXAMPLE)

#### Recent progress in C++ in this area
The C++ crowd is working on this. 
Ref: https://techblog.rosemanlabs.com/c++/safety/object-lifetime/2025/08/28/a-safe-pointer-that-protects-against-use-after-free-and-updates-when-the-pointee-is-moved.html



## A simple example - a tree with back links.
(MORE)

### True multiple ownership with weak back references
This is the more general case, where reference counts are doing real work.

(MORE)


#### A complex example -- bidrectional transitive closure with many back references
This is a program previously posted on the Rust Forums for code review. 

Ref to running code: [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=3071faba427b440643d26ac5fe182caa)
If you run this with cargo test -- --nocapture you can see the output. Playground doesn't seem to offer that option, unfortunately.

This is a bidrectional transitive closure algorithm. What it's doing is computing whether you can get there from here for a big grid of rectangular regions.
If two rectangles share some edge space, they're reachable from each other.
The algorithm computes collections of VisGroup, regions which are all reachable from each other.
It's code used in a real application, not just a toy example.
It's more complex than the usual doubly-linked list and backlinked tree examples used for this sort of thing.

The trick is doing this entirely in safe Rust. It uses RefCell, borrow() calls, and Weak pointers for its non-owning back pointers.
As is typical with Rust, it was really hard to get the borrow plumbing right, and then it just worked.

The basic data structure here is a set whose members can find the set of which they are a part. Basic set operations, such as union, are provided.
Back references from elements to sets are updated when a union operation is performed.

The goal here is to show how all **.borrow()** and **.borrow_mut()** calls in this code could be checked at compile time. 
We can then replace all **RefCell<T>** instances with our proposed **CompiledRefCell<T>**

This code mostly reliies on disjointness by type and disjointness by scope. But there is a place where disjointness by instance is required.

    for weak_block in &self_shared_groups {
        if !Weak::ptr_eq(&self.weak_link_to_self, weak_block) {
            if let Some(block) = weak_block.upgrade() {
                block.borrow_mut().viz_group = self.viz_group.clone();
            }
        }
    }

The run time **ptr_eq** check establishes that weak_block and weak_link_to_self are disjoint.
Within the scope of that test, we can thus safely borrow both simultaneously.

Question: are there other situations where a different kind of check that instances are disjoint is necessary?

(MORE)

(Use case: this is for a support tool for my metaverse client for Second Life/Open Simulator. It is a rule of the grid that you can only see regions you can reach. Some areas are full of checkerboards of isolated private regions, none of which can see each other. Others are big land areas, sometimes connected only by narrow strings of regions. I'm working on an impostor system for long-distance visibility, and I need to do some pre-computation to decide which impostors are visible from where.)

## Trouble spots
### Statics
(MORE)
### Generics and traits
These make it hard to trace the call tree.

(MORE)
### Function references/lambdas?
(MORE)

# In-progress notes to be merged into the above
The preferred coding style is short borrows with very narrow scope, typically within one statement. That's normally considered bad, because it means extra time checking for double borrows. But if this kind of borrow can be made compile-time, it's free. The compiler is going to have to check for overlapping borrow scopes, and the more local the borrow scope, the less compile time work is involved. Storing a borrowed link in a structure or a static requires too much compile-time analysis, but if the lifetime of the borrow is very local, the analysis isn't hard. Borrow scopes which include a function call imply the need for cross-function call tree analysis. It's been pointed out to me that traits and generics make this especially hard, because you don't know what the instantiation can do before generics are instantiated. That might require some kind of annotation that says to the compiler "Implementations of function foo for trait Foobar are not allowed to do an upgrade of an Rc<Bar>". Then, at the call, the compiler can tell it's OK for a borrow scope to include that call, and in an implementation of the called function, upgrade of Rc<Bar> is prohibited. In fact, if this is done for all functions that receive an Rc<Bar> parameter, it might be possible to do all this without call tree analysis. It's becomes a type constraint. Need to talk to the type theorists, of which I am not one.

All those .borrow() calls are annoying. It might be possible to elide them most of the time, in the same way that you can refer to the item within an Rc without explicitly de-referencing the Rc. That would be nice from an ergonomic perspective. But it runs the risk of generating confusing compiler messages when a problem is detected. It would be like those confusing error messages that arise when type inference can't figure something out or the borrow checker needs explicit lifetime info to resolve an ambiguity.



