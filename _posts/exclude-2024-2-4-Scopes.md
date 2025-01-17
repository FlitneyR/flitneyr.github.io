---
title: Resource Scopes - A beautifully simple ownership model
layout: post
---

In the process of writing a Vulkan-based game engine, I've come across a beautifully simple ownership model
that's turbo charged my progress. I'm referring to it here as a `ResourceScope`, but it's similar in concept to Arenas.
The implementation is very simple:

{% highlight c++ linenos %}

class ResourceScope {
    std::list<std::function<void()>> m_deferredCleanupCommands;

    // ... deleted constructors

public:
    ResourceScope() {}

    ~ResourceScope() {
        executeDeferredCleanupFunctions();
    }

    void addDeferredCleanupFunction(std::function<void()> func) {
        m_deferredCleanupCommands.push_back(func);
    }

    void executeDeferredCleanupFunctions() {
        while (!m_deferredCleanupCommands.empty()) {
            m_deferredCleanupCommands.back()();
            m_deferredCleanupCommands.pop_back();
        }
    }
};

{% endhighlight %}

And using it is also simple: whenever you create a resource, add a command to delete it later. For example:

{% highlight c++ linenos %}

vk::ResultValue<vk::Pipeline> GraphicsPipelineBuilder::build() {
    // ...

    vk::ResultValue<vk::Pipeline> pipelineResult = m_device.createGraphicsPipeline(
        VK_NULL_HANDLE,
        createInfo
    );

    if (pipelineResult.result != vk::Result::eSuccess)
        return vk::ResultValue<vk::Pipeline> { pipelineResult.result, VK_NULL_HANDLE };

    r_scope.addDeferredCleanupFunction([=, device = m_device, pipeline = pipelineResult.value]() {
        device.destroyPipeline(pipeline);
    });

    // ...
}

{% endhighlight %}

So this is pretty simple. Why do I like it so much? First off: functional programming! Woop!
But seriously, trying to solve scope problems from an OOP perspective is pretty messy and often
ends in existential questions of ownership, and they end up very much tied to object life times,
which, at best might not be the best way to represent scope, and at worst might lead to you destroying
resources before others which depend on them (this might be fine most of the time, especially if
destroying a resource ≈ freeing some memory, but it's against the Vulkan spec to destroy a resource
that is depended on by another un-freed resource).

With `ResourceScope`s, so long as you add clean-up functions as you create resources, you are guaranteed not to destroy resources before
others that depend on them, because to do that you'd have to create them out of order.

But the biggest benefit of this method, is that it moves ownership and scope and resource management from a compile
time decision, to a runtime decision. Ownership decisions often lock you into certain designs: "A material owns a pipeline, pipeline layout,
shader modules, descriptor sets, descriptor set layouts and textures" might make sense when you start making your first material class.
But then you realise that most materials reuse the same pipeline layout, and the same pipeline, and the same shader modules, and the same
descriptor set layouts, and only differ by descriptor sets and textures. So maybe you refactor and move ownership of those elements out
of your material class and leave managing their lifetime to the user of your material class. But maybe the user really doesn't care about
that, they only need one material so they can't benefit from reusing resources, and they don't want to have to worry about the shader modules
and descriptor sets or whatever. Resource scopes let the user of your class create those resources (through e.g. a builder class), and forget
about them.

As an example of the difference this made, this project has a builder class to simplify making Vulkan pipelines:
`GraphicsPipelineBuilder : IBuilder<vk::ResultValue<vk::Pipeline>>`. Before adding `ResourceScope`s, it had a different type:
`GraphicsPipelineBuilder : IBuilder<vk::ResultValue<PipelineData>>`, where `PipelineData` stored various data about
a pipeline (shocking, I know).

{% highlight c++ linenos %}

struct PipelineData {
    std::vector<vk::ShaderModule> shaderModules;
    vk::PipelineLayout            pipelineLayout;
    vk::Pipeline                  pipeline;
};

{% endhighlight %}

So ... who owns these? Who should be responsible for destroying them? Pipeline layouts and shader modules are commonly shared
between multiple pipelines. I don't have a good answer for any of this, so I can't make any smart decisions about this data.
So I just pass the data out and leave the user to manage freeing these resources. Not very good.

You could wrap this stuff up in reference counted pointers, like `std::shared_ptr`s for instance, with a destructor that
frees their resources. That solves one problem: shared ownership. Which is a good start, but it doesn't address order of
destruction and makes reasoning about lifetimes harder. It doesn't really solve the problem, it just moves it.

The overarching problem though is that we're still thinking about ownership, when we should be thinking about scope.
To incorporate `ResourceScope`s took a few steps. First of all, the builder needs to keep track of what scope it is working in.
So in `GraphicsPipelineBuilder`'s base class - `IBuilder` - we make sure to store a reference to the working scope.

{% highlight c++ linenos %}

template<typename T>
class IBuilder {
    ResourceScope& r_scope;

public:
    IBuilder(ResourceScope& scope) : r_scope(scope) {}

    T build();
};

{% endhighlight %}

Then every time you create a resource which needs to be cleaned up later, you add a deferred clean up command to the scope:

{% highlight c++ linenos %}

vk::ShaderModule IPipelineBuilder::loadShaderModule(const char* filename) {
    // ...

    vk::ShaderModule shaderModule = m_device.createShaderModule(/* ... */);
    
    r_scope.addDeferredCleanupFunction([=, device = m_device]() {
        device.destroyShaderModule(shaderModule);
    });

    return shaderModule;
}

// ...

vk::ResultValue<vk::Pipeline> GraphicsPipelineBuilder::build() {
    // ...

    vk::ResultValue<vk::Pipeline> pipelineResult = m_device.createGraphicsPipeline(
        VK_NULL_HANDLE,
        createInfo
    );

    if (pipelineResult.result != vk::Result::eSuccess)
        return vk::ResultValue<vk::Pipeline> { pipelineResult.result, VK_NULL_HANDLE };

    r_scope.addDeferredCleanupFunction([=, device = m_device, pipeline = pipelineResult.value]() {
        device.destroyPipeline(pipeline);
    });

    return vk::ResultValue<vk::Pipeline> { vk::Result::eSuccess, pipelineResult.value };
}

{% endhighlight %}

This drastically simplifies managing these resources, while still allowing fairly granular control to the user.
You can choose which resource scope you want to use for certain resources and have them destroyed at a time of your choice.
And you can have different scopes for different resources, so you can create the shader modules that you want to reuse in one
scope. Then specific resources that depend on them in another resource scope. It is important if you do that, to make sure that the
resource scopes are nested, so that you don't for example destroy a shader module before destroying a pipeline that depends on it.

But that's also pretty simple:

{% highlight c++ linenos %}

ResourceScope outerScope;

// ... add stuff to outer scope

ResourceScope nestedScope;
outerScope.addDeferredCleanupCommand([&]() { nestedScope.executeCleanupCommands(); });

// ... add stuff to inner scope that depends on the outer scope

{% endhighlight %}

Now you can call `nestedScope.executeCleanupCommands()` whenever you like to free those resources, or you can call
`outerScope.executeCleanupCommands()` which will call `nestedScope.executeCleanupCommands()` before cleaning up anything that
was added to `outerScope` before `nestedScope`, ensuring that you won't destroy any resources before other resources that depend
on them.

Now a more interesting example, staging buffers:

{% highlight c++ linenos %}

/**
 * @brief Copies data via a staging buffer
 * 
 * @param fence The fence to signal for an asynchronous staged copy.
 *  If one is provided, the function will be called asynchronously
 * @param signalSemaphore The semaphore to signal for an asynchronous staged copy.
 *  If one is provided, the function will be called asynchronously
 */
vk::Result Allocated<vk::Buffer>::stagedCopyData(const void* data, uint32_t size, vk::Fence fence, vk::Semaphore signalSemaphore) {
    // check if the user wants to manage synchronisation through a fence or semaphore
    bool async = fence != VK_NULL_HANDLE || signalSemaphore != VK_NULL_HANDLE;

    // create a resource scope for any temporary resources we need
    // using a pointer here so it can be deleted from another thread if we execute asynchronously
    ResourceScope* tempScope = new ResourceScope();

    IEngine& engine = IEngine::get();
    vk::Device device = engine.getDevice();

    // if the user didn't provide a fence, we need to make a temporary one
    if (fence == VK_NULL_HANDLE) {
        fence = device.createFence(vk::FenceCreateInfo {});
        tempScope->addDeferredCleanupFunction([=]() { device.destroyFence(fence); });
    }

    // create a staging buffer in local temporary scope and copy the data to it
    vk::ResultValue<Allocated<vk::Buffer>> stagingBuffer = BufferBuilder { *tempScope }
        .setAllocationUsage(VMA_MEMORY_USAGE_CPU_TO_GPU)
        .setBufferUsage(vk::BufferUsageFlagBits::eTransferSrc)
        .setSizeBuildAndCopyData(data, size);
    
    if (stagingBuffer.result != vk::Result::eSuccess) {
        delete tempScope;
        return stagingBuffer.result;
    }

    // copy from the staging buffer to the m_inner buffer
    vk::CommandBuffer cmd = engine.beginOneTimeCommandBuffer(vkb::QueueType::graphics);
    cmd.copyBuffer(*stagingBuffer.value, m_inner, vk::BufferCopy {}.setSize(size));

    // signal the semaphore if one was provided
    vk::SubmitInfo submitInfo;
    if (signalSemaphore != VK_NULL_HANDLE) submitInfo.setSignalSemaphores(signalSemaphore);

    engine.submitOneTimeCommandBuffer(cmd, vkb::QueueType::graphics, submitInfo, fence);

    if (async) {
        // start a thread to asynchronously wait for the copy to finish and then clean up temporary resources
        std::thread cleanUpAndSignalFence ([=]() {
            device.waitForFences(fence, true, UINT64_MAX);
            delete tempScope;
        });
        cleanUpAndSignalFence.detach();

        return vk::Result::eSuccess;
    } else {
        // synchronously wait for the copy to finish and then clean up temporary resources
        vk::Result result = device.waitForFences(fence, true, UINT64_MAX);
        delete tempScope;
        return result;
    }
}

{% endhighlight %}

Here, we can easily adapt to different ownership models at runtime, in one function, just depending on whether the user provided any
synchronisation resources. We don't need to track whether we created the fence, or if the user provided it. And we can defer completion
of this function to another thread to accommodate asynchronous GPU operations, while still providing full control to whoever uses this
function as to how they want to synchronise its work: fully synchronous by not providing a fence or semaphore, wait on the CPU by providing
a fence and no semaphore, or wait on the GPU by providing a semaphore.

I've just started experimenting, and this simple little class is already making this project go a whole lot smoother
than my previous attempt.

My only reservation is the extensive use of `std::function`. I'm not going to make any performance claims until I've
benchmarked this but it would almost certainly be faster to use some kind of class with a virtual function for this.
