# On a taxonomy of renderers

*[This originally appeared on Hacker News, in a shorter form.](https://news.ycombinator.com/item?id=43908220)*

This is a problem I've had to think about too much.
As a heavy use of the Rust graphics stack, I get to see people struggling with how to approach the rendering engine problem. There' a whole stack of components. What each component should do remains a problem, and APIs are difficult.
There's even a question of whether there should be "rendering engines" as components at all, as opposed to integrated game engines.

Here's the Rust stack:

- Down at the bottom is the GPU hardware.

- Then there's the GPU driver. This can come from the hardware vendor, or from a third party. There's an open source driver for NVidia GPUs. The APIs offered by GPU drivers are usually Vulkan, Direct-X, Metal, and/or OpenGL. This level has settled down and works reasonably well. Vulkan's API is more like a parts kit that exposes most of the GPU's functionality without abstracting it much.

- Next is a sanity, safety, and resource allocation layer. This layer doesn't even have a good name. It's built into OpenGL, but it has to be explicitly implemented for Vulkan. The GPU has its own memory, and Vulkan has a limited allocator which lets you request large GPU memory buffers. You need a second allocator on top of that one to allocate arbitrary-sized memory buffers. There are other resources - queues, bindless descriptor slots, and such - all of which have to be allocated. There are synchronization issues. How the CPU talks to GPU memory is totally different for "integrated" GPUs as usually seen in laptops, where the GPU lacks dedicated memory. There are various modes where CPU and GPU can access the same memory with a performance penalty, or, alternatively, messages are passed back and forth over queues and the GPU's DMA hardware does the transfer. There are also extra restrictions for phone OSs and web browsers, which are much more constrained than desktop. Vulkan offers considerable parallelism, which the higher levels should use to get good performance. Vulkan has very complex locking constraints, and something has to check that they are not violated.

This layer is "WGPU" in the Rust stack. The API WGPU offers to higher levels looks a lot like Vulkan, but there's a lot of machinery in there which exists mostly to resolve the above issues.

- Next step up is the "rendering engine". This is where you put in meshes, textures, lights, and material data. You provide transforms to place the meshes, and camera information. The rendering engine puts the scene on the screen. Rendering engines are straightforward until you get to lighting, shadows, and big dynamic scenes. Then it gets hard. In the Rust world, the easy part of this has been done at least five times, but nobody has done the hard part yet.

There's an argument that you shouldn't even try. High-performance lighting and shadows require info about which lights can illuminate which meshes. A naive implementation forces the GPU to test every light against every object, which will slow rendering to a crawl when there are many lights. So the renderer needs info about what's near what.

That kind of spatial information usually belongs to the scene graph, which is part of the next level up, the "game engine". So how does the renderer get that info? Being explicitly told by the game engine? Callbacks to the game engine? Computing this info when the scene changes and caching it? Or should you give up on a general purpose rendering engine and integrate rendering into the game engine, which usually has better spatial data structures. The big game engines such as UE mostly integrate the rendering and game engines, which allows them to pre-compute much info in the scene editor during level design. That's UE's big trick - do everything possible when building the content, rather than deferring work to run time. Unreal Engine Editor does much of the heavy lifting. The work of the "rendering engine" is divided between the run-time player, the run-time game engine, and the build-time editor and optimizer.

Worlds with highly dynamic user-provided content can't do that kind of precomputation. This is a "metaverse problem", and is one of the reasons metaverses have scaling problems. (Meta spend $40 billion, and Improbable spent $400 million trying to do a metaverse. This is part of why.)

Now that's a taxonomy issue. I would have liked to see more insight in the paper on where to cut the layers apart and what they need to say to each other. The authors have a point that the industry doesn't talk about this enough. As I noted above, we don't even have good names for some of this stuff.


