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

### Checking rules
This is the hard part. Borrows must be disjoint. They can be
* Disjoint as to type - RefCell<A> and RefCell<B>, where A and B cannot be the same type, are disjoint.
* Disjoint as to scope - if all **.borrow()** and **.borrow_mut()** calls return results with statically defined scopes, and no two scopes of the same type overlap, the borrows are disjoint.
* Disjoint as to instance - if two items are both borrowed in overlapping scopes, but the borrowed objects can be shown to be different objects, the borrows are disjoint. This is the hard one. If such disjointness analysis is confined to single functions, the checker's job appears manageable.


## A simple example - a tree with back links.
(MORE)

## A complex example -- bidrectional transitive closure with many back references
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



--------------------
## Outline
### Notes on compile time borrow checking for RefCell type situations

- Goal: Do at compile time what RefCell / .borrow_mut() does at run time.
- Goal: figure out how far we can get with somewhat simple checking.

- Disjointness in code
-- No two .borrow_mut() calls for the 'same RefCell' at the same time.
--- Mostly enforced by scope.
--- Requires tracing the call chain.
--- Generics can limit tracing.

- Disjointness in data.
-- What does "same RefCell" mean?
--- RefCell<T>, where T is different, is usually not the 'same RefCell'.
---- Are there exceptions to this?

- If either code or data demonstrate disjointness, it's safe.
-- Exceptions?

- Trouble spots.
-- Statics
-- Generics
-- Function references/lambdas?

- Use cases
-- Given a tree with backpointers, can we swap two subtrees?
--- Hard, but if we can do that, this is useful.
--- Can we have two .borrow_mut() scopes active on two
different RefCells of the same type?
--- When can we easily tell those are disjoint?
---- They're both borrowed in the same function,
from structs we know are disjoint.
---- This is the hard case.

- Basic question: is there an intersection between what we can check easily and what is useful?

struct Node {
   parent: Option<Rc::Weak<RefCell<Node>>>,
   children: Vec<Rc<RefCell<Node>>>,
}
fn swap_subtree(&mut node1: Node, child1: usize, &mut Node2, child2: usize)
{
}

If we can make that work, this is useful.

