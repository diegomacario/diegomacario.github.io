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

Now it's impossible to forget to delete the OpenGL objects, which is great, but looking at the code above you might think:

> Why doesn't the `LoadMeshFromFile` function return a `Mesh` object instead of three separate values? By returning a `Mesh` object there's no chance that somebody would forget to wrap the values it returns.

And you are totally right, that's much cleaner. With that change, the `LoadMeshFromFile` function would look like this:

{% highlight cpp %}
Mesh LoadMeshFromFile(const std::string& filePath)
{
    unsigned int VAO, EBO, numIndices;

    // Read the vertex attributes from the glTF file
    // Configure the VAO and the EBO

    // Wrap the OpenGL objects
    Mesh localMesh(VAO, EBO, numIndices);

    return localMesh;
}
{% endhighlight %}

And using it would look like this:

{% highlight cpp %}
Mesh teapot = LoadMeshFromFile("assets/models/teapot.gltf");
teapot.render();
{% endhighlight %}

Everything looks good, but unfortunately the code above has a subtle problem that will cause it to fail. To understand that problem, let's do a step-by-step walkthrough of this line of code:

{% highlight cpp %}
Mesh teapot = LoadMeshFromFile("assets/models/teapot.gltf");
{% endhighlight %}

1. The `LoadMeshFromFile` function returns a `Mesh` object by value. That temporary `Mesh` object (let's refer to it as `tempMesh`) is copy-constructed from `localMesh`. This means that at this point we have two objects (`localMesh` and `tempMesh`) that wrap the same OpenGL objects.

2. When we exit `LoadMeshFromFile`, `localMesh` goes out of scope, which causes the OpenGL objects it wraps to be deleted in its destructor. Since `tempMesh` wraps the same OpenGL objects, this means that the objects that `tempMesh` wraps have now been deleted too.

3. Finally, the `teapot` object is copy-constructed from `tempMesh`. This means that `teapot` ends up wrapping the already deleted objects too.

So the call to `teapot.render();` will fail because the VAO and the EBO of the teapot have already been deleted.

To understand that problem, let's go back to the definition of the `Mesh` class. We only defined a constructor, a destructor and the `render` method in that class, but C++ silently defined two more methods: the copy constructor and the copy assignment operator. Those methods look like this:

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



[Jekyll docs][jekyll-docs]

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
