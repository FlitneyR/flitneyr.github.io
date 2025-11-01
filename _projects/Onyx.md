---
layout: post
title: "Onyx"
date: 2025-11-01
priority: 0
---

Another Vulkan/ECS game engine - this time with an editor

In this project, I tried to apply what I've learnt in the past year to make better decisions at every turn.
I focused, early on, on laying out a plan for the overall project structure.

A game written in Onyx would consist of four primary components:
- The core engine code
- The shared game logic code
- The game's editor
- The game itself

The ECS is more split apart than in previous projects, to allow for more flexibility. It also allows for specific
context data to be sent to various update stages, rather than that type of data being stored as singletons in the ECS world itself.

I made more use of runtime checks for things like hashes of type names, rather than the excessive templating that was present
in some of my previous ECS engines.

I have also written up the following articles outlining some of Onyx's features:
- [Onyx's ECS. From the top down, and the bottom up]({% post_url 2025-11-01-OnyxECS %})

Below is a quick demo, poking around the editor, then showing a simple game running.

<iframe width="560" height="315" src="https://www.youtube.com/embed/eptxYNMaZvs?si=OekyU9MLDDMdmwKU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

[Show me the code](https://www.github.com/FlitneyR/onyx)
