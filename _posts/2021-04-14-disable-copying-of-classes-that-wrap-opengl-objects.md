---
layout: post
title: "Always remember to disable the copying of classes that wrap OpenGL objects"
tags: C++ OpenGL
---

One thing that really surprised me when I started working with OpenGL is that when you write a class that wraps an OpenGL object, you should never copy any instances of that class. To understand why, consider this example:

Let's say you have a function called `LoadMeshFromFile` that reads the vertex attributes of a mesh from a glTF file and returns a VAO and an EBO that can be used to render that mesh:

{% highlight cpp %}
// The last three parameters are output parameters
void LoadMeshFromFile(const std::string& inFilePath,
                      unsigned int& outVAO,
                      unsigned int& outEBO,
                      unsigned int& outNumIndices);
{% endhighlight %}

Using that function to load a teapot and render it would look like this:

{% highlight cpp %}
// Load the teapot
unsigned int VAO, EBO, numIndices;
LoadMeshFromFile("assets/models/teapot.gltf", VAO, EBO, numIndices);

// Render it
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, numIndices, GL_UNSIGNED_INT, 0);
glBindVertexArray(0);

// Delete it
glDeleteVertexArrays(1, &VAO);
glDeleteBuffers(1, &EBO);
{% endhighlight %}

Note the calls to delete the VAO and the EBO at the bottom. We need to remember to do that once we are done working with the teapot, or otherwise we would leak those OpenGL objects. The problem is that that's something that's easy to forget, but we can make sure that we always do it by using this simple wrapper class:

{% highlight cpp %}
class Mesh
{
public:
    Mesh(unsigned int VAO, unsigned int EBO, unsigned int numIndices)
        : mVAO(VAO), mEBO(EBO), mNumIndices(numIndices)
    {

    }

    ~Mesh() {
        // Delete the OpenGL objects when instances of this wrapper class are destroyed
        glDeleteVertexArrays(1, &mVAO);
        glDeleteBuffers(1, &mEBO);    
    }

    void render() {
        glBindVertexArray(mVAO);
        glDrawElements(GL_TRIANGLES, mNumIndices, GL_UNSIGNED_INT, 0);
        glBindVertexArray(0);
    }

private:
   unsigned int mVAO, mEBO, mNumIndices;
};
{% endhighlight %}

With the `Mesh` class in our toolbox, loading the teapot and rendering it would now look like this:

{% highlight cpp %}
// Load the teapot
unsigned int VAO, EBO, numIndices;
LoadMeshFromFile("assets/models/teapot.gltf", VAO, EBO, numIndices);

// Wrap the OpenGL objects
Mesh teapot(VAO, EBO, numIndices);

// Render the teapot
teapot.render();

// When the teapot variable goes out of scope,
// its destructor automatically deletes the OpenGL objects
{% endhighlight %}

Now it's impossible to forget to delete the OpenGL objects, but looking at the code above you might think:

> Why doesn't the `LoadMeshFromFile` function return a `Mesh` object instead of three separate values? By returning a `Mesh` object there's no chance that somebody would forget to wrap the values it returns.

And you are totally right, that's much cleaner. With that change, the `LoadMeshFromFile` function would look like this:

{% highlight cpp %}
Mesh LoadMeshFromFile(const std::string& filePath)
{
    unsigned int VAO, EBO, numIndices;

    // Read the vertex attributes from the glTF file
    // Configure the VAO and the EBO

    Mesh loadedMesh(VAO, EBO, numIndices);

    return loadedMesh;
}
{% endhighlight %}

And using it would look like this:

{% highlight cpp %}
Mesh teapot = LoadMeshFromFile("assets/models/teapot.gltf");
teapot.render();
{% endhighlight %}

Everything seems perfect, but unfortunately the code above has a subtle problem that will cause it to fail. To understand that problem, let's do a step-by-step walkthrough of this line of code:

{% highlight cpp %}
Mesh teapot = LoadMeshFromFile("assets/models/teapot.gltf");
{% endhighlight %}

- The `LoadMeshFromFile` function returns a `Mesh` object by value. That temporary `Mesh` object (let's refer to it as `returnedTempMesh`) is copy-constructed from `loadedMesh`:

{% highlight cpp %}
// returnedTempMesh is the Mesh that's returned by this function
Mesh LoadMeshFromFile(const std::string& filePath)
{
    // ...

    // returnedTempMesh is copy-constructed from loadedMesh
    return loadedMesh;
}
{% endhighlight %}

- This means that at one point in time we have two objects (`loadedMesh` and `returnedTempMesh`) that wrap the same OpenGL objects.

- When we exit `LoadMeshFromFile`, `loadedMesh` goes out of scope, which causes the OpenGL objects it wraps to be deleted in its destructor. Since `returnedTempMesh` wraps the same OpenGL objects, this means that the objects that `returnedTempMesh` wraps have now been deleted too.

- Finally, the `teapot` object is copy-constructed from `returnedTempMesh`, which causes it to end up wrapping the already deleted OpenGL objects too.

So the call to `teapot.render();` will fail because the VAO and the EBO of the teapot have already been deleted by the time we execute it.

The root of this problem is the copy constructor and the copy assignment operator that C++ silently generated for us in the `Mesh` class. Those methods look like this:

{% highlight cpp %}
// Copy constructor
Mesh::Mesh(const Mesh& rhs)
    : mVAO(rhs.mVAO)
    , mEBO(rhs.mEBO)
    , mNumIndices(rhs.mNumIndices)
{

}

// Copy assignment operator
Mesh& Mesh::operator=(const Mesh& rhs) {
    mVAO = rhs.mVAO;
    mEBO = rhs.mEBO;
    mNumIndices = rhs.mNumIndices;
    return *this;
}
{% endhighlight %}

As OpenGL developers we know that even though `mVAO` and `mVBO` are `unsigned ints`, those two member variables effectively behave as pointers to memory on the GPU, which is why we would never copy them. The problem is that C++ doesn't know that. In its eyes `mVAO` and `mVBO` are simple `unsigned ints` that can be easily copied. If they were actual pointers, C++ wouldn't generate a copy constructor and a copy assignment operator for us:

{% highlight cpp %}
class Mesh
{
public:
    // ...

private:
    // When an instance of this class is copied, should the mVAO and mEBO of the copy
    // point to the same VAO and EBO as the original? Or should the VAO and EBO be
    // duplicated in the GPU so that each instance can point to its own OpenGL objects?
    // C++ can't make that decision, so it wouldn't generate a copy constructor and
    // a copy assignment operator for us
    VAO* mVAO;
    EBO* mEBO;
    unsigned int mNumIndices;
};
{% endhighlight %}

So how do we fix this? The first solution that comes to mind is to define the copy constructor and the copy assignment operator of the `Mesh` class ourselves, and to somehow copy the OpenGL objects in them so that when they are called each `Mesh` object involved ends up wrapping its own OpenGL objects:

{% highlight cpp %}
// Copy constructor
Mesh::Mesh(const Mesh& rhs)
{
    // Somehow duplicate rhs.mVAO and rhs.mEBO in the GPU

    // Wrap the duplicates
    mVAO = copyOfVAO;
    mEBO = copyOfEBO;
    mNumIndices = rhs.mNumIndices;
}

// Copy assignment operator
Mesh& Mesh::operator=(const Mesh& rhs) {
    // Somehow duplicate rhs.mVAO and rhs.mEBO in the GPU

    // Wrap the duplicates
    mVAO = copyOfVAO;
    mEBO = copyOfEBO;
    mNumIndices = rhs.mNumIndices;
    return *this;
}
{% endhighlight %}

That would be really nice, but according to the OpenGL documentation it's not a good idea:

> Copying an OpenGL object's data to a new object is incredibly expensive; it is also essentially impossible to do, thanks to the ability of extensions to add state that you might not statically know about.

So the only option that we are left with is to disable the copying of the `Mesh` class so that nobody runs into this problem while using it. It may seem hard to work with `Mesh` objects that cannot be copied, but remember that they can still be moved if we define a move constructor and a move assignment operator. With those final improvements, the `Mesh` class looks like this:

{% highlight cpp %}
class Mesh
{
public:
    // ...

    // Delete the copy constructor and the copy assignment operator so that
    // instances of this class cannot be copied anymore
    Mesh(const Mesh& rhs) = delete;
    Mesh& operator=(const Mesh& rhs) = delete;

    // Move constructor
    Mesh(Mesh&& rhs) noexcept
        : mVAO(std::exchange(rhs.mVAO, 0))
        , mEBO(std::exchange(rhs.mEBO, 0))
        , mNumIndices(std::exchange(rhs.mNumIndices, 0))
    {

    }

    // Move assignment operator
    Mesh& operator=(Mesh&& rhs) noexcept
    {
       mVAO = std::exchange(rhs.mVAO, 0);
       mEBO = std::exchange(rhs.mEBO, 0);
       mNumIndices = std::exchange(rhs.mNumIndices, 0);
       return *this;
    }

    // ...

private:
   unsigned int mVAO, mEBO, mNumIndices;
};
{% endhighlight %}

Now it's impossible to copy `Mesh` objects. They can only be moved, and when a move occurs, the `mVAO` and `mVBO`of the object that we move from are set equal to 0, which is equivalent to saying that it doesn't wrap any OpenGL objects anymore.

And something really nice is that we don't have to make any changes to the `LoadMeshFromFile` function. To understand why, let's do a step-by-step walkthrough of the same line of code as before:

{% highlight cpp %}
Mesh teapot = LoadMeshFromFile("assets/models/teapot.gltf");
{% endhighlight %}

- The `LoadMeshFromFile` function returns a `Mesh` object by value. That temporary `Mesh` object (let's refer to it as `returnedTempMesh`) is **move-constructed** from `loadedMesh`:

{% highlight cpp %}
// returnedTempMesh is the Mesh that's returned by this function
Mesh LoadMeshFromFile(const std::string& filePath)
{
    // ...

    // returnedTempMesh is move-constructed from loadedMesh
    return loadedMesh;
}
{% endhighlight %}

- This means that when `returnedTempMesh` is move-constructed, `loadedMesh` stops wrapping any OpenGL objects.

- When we exit `LoadMeshFromFile`, `loadedMesh` goes out of scope, which causes its destructor to be called. Since its `mVAO` and `mVBO` were set to 0 during the move-construction of `returnedTempMesh`, its destructor doesn't delete any OpenGL objects.

- Finally, the `teapot` object is **move-constructed** from `returnedTempMesh`. Because of this, when `returnedTempMesh` is destroyed, it doesn't delete the OpenGL objects that `teapot` now wraps.

I know this all seems terribly complicated at first, but once you know the rules of the game you can focus on actually playing it and on the things that make it fun, like rendering beautiful teapots!

<p align="center">
<img src="/assets/images/teapot.png" alt="Colourful teapot" width="728"/>
</p>
