(Work in progress)

## Bidrectional transitive closure with many back references

September, 2025
John Nagle

This is a program posted on the Rust Forums for discussion.

Ref: [https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=3071faba427b440643d26ac5fe182caa](Rust Playground)

This is a bidrectional transitive closure algorithm. What it's doing is computing whether you can get there from here for a big grid of rectangular regions. If two rectangles share some edge space, they're reachable from each other. The algorithm computes collections of VisGroup, regions which are all reachable from each other.

The trick is doing this entirely in safe Rust. It uses RefCell, borrow() calls, and Weak pointers for its non-owning back pointers. As is typical with Rust, it was really hard to get the borrow plumbing right, and then it just worked.

So was this the right way to organize this algorithm? Comments?

If you run this with cargo test -- --nocapture you can see the output. Playground doesn't seem to offer that option, unfortunately.

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

