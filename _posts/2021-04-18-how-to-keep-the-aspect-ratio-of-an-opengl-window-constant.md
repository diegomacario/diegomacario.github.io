---
layout: post
title: "How to keep the aspect ratio of an OpenGL window constant when it's resized"
tags: C++ OpenGL
---

Something that I had a hard time figuring out when I started working with OpenGL is how to keep the aspect ratio of a window constant when it's resized. For some reason I couldn't find information about that topic anywhere. Hopefully this post will help you if you are as lost as I was back then.

Note that I will use the [GLFW](https://www.glfw.org/) OpenGL library in my explanation because that's my favourite library for creating windows, but you can use the ideas that I will present here with any other library.

Below is a screenshot of an application that I finished writing recently. When you first launch it, its window has a width of `1280` pixels and a height of `720` pixels. That means that it has an aspect ratio of 
`1280 / 720`, or  `1.777`.

<p align="center">
<img src="/assets/images/constant_aspect_ratio/original_ar.PNG" alt="Original aspect ratio" width="600"/>
</p>

If I grab the right edge of the window and drag it towards the right, look at what happens to the image:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/bad_ar_horizontal.PNG" alt="Bad horizontal aspect ratio" width="728"/>
</p>

Ugh, that reminds me of the PowerPoint presentations that I used to make in third grade. The same is the case if I grab the bottom edge of the window and drag it downwards:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/bad_ar_vertical.PNG" alt="Bad vertical aspect ratio" width="600"/>
</p>

The unpleasant stretching you see above is happening because we are allowing the aspect ratio of the image to change. The code that enables that behaviour is really simple. This is how I create the GLFW window:

{% highlight cpp %}
// Initialize GLFW
glfwInit();

// Create a window with a width of 1280 screen coordinates, a height of 720
// screen coordinates and a title of "Hello, World!"
GLFWwindow* window = glfwCreateWindow(1280, 720, "Hello, World!", nullptr, nullptr);

// Make the OpenGL context of our window current
glfwMakeContextCurrent(window);

// Load pointers to OpenGL functions using GLAD
gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);

// Set the framebuffer resize callback, which is called when
// the framebuffer of our window is resized
glfwSetFramebufferSizeCallback(window, framebufferSizeCallback);
{% endhighlight %}

The cause of the stretching is the `framebufferSizeCallback` function, which I defined like this:

{% highlight cpp %}
// The signature of this function comes from the GLFW documentation
// It's required to be this way so that we can pass it to glfwSetFramebufferSizeCallback
void framebufferSizeCallback(GLFWwindow* window,
                             int widthOfFramebuffer,
                             int heightOfFramebuffer)
{
    // Update the dimensions of the viewport so that it covers the entire framebuffer
    // The first two parameters are the X and Y coordinates of the lower left corner of
    // the viewport, in pixels
    // The last two parameters are the width and height of the viewport, in pixels
    glViewport(0, 0, widthOfFramebuffer, heightOfFramebuffer);
}
{% endhighlight %}

To understand why that function results in stretching, let's do a step-by-step walkthrough of what happens when you grab the right edge of the window and you change its dimensions from `1280 x 720` to `1440 x 720`:

- GLFW is notified of the change in dimensions, which causes it to resize the front and back framebuffers of the window to cover its new area. Note that those are the framebuffers that you draw into in your render loop, and that you clear and swap with calls like `glClear(GL_COLOR_BUFFER_BIT)` and `glfwSwapBuffers(window)`.

- After the framebuffers have been resized, the `framebufferSizeCallback` function is executed. In that function we call `glViewport` with the new dimensions of the framebuffers. Remember that `glViewport` is used to specify the area of the framebuffers that we want to use for drawing. By passing it the new dimensions of the framebuffers we are simply saying that we want to use their entire area for drawing. This causes the aspect ratio of our frames to become `1440 / 720`, or `2.0`.

- That increase in the aspect ratio causes our frames to be stretched horizontally.

So how do we fix this? In short, all we need to do is find the biggest area that we can fit inside the framebuffers that has the original aspect ratio of `1.777`, and then we need to call `glViewport` to specify that area.

To understand why that works, think about it this way: we can achieve an aspect ratio of `1.777` with endless combinations of widths and heights. Some examples include `1056 x 594`, `1280 x 720` and `1600 x 900`. By using the biggest of those combinations that fits, our frames will end up covering as much space as possible in the framebuffers and they will have the correct aspect ratio.

At this point you are probably wondering what the results of the fix I described above look like. Here is what happens when I grab the right edge of the window and drag it towards the right:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_horizontal.PNG" alt="Good horizontal aspect ratio" width="728"/>
</p>

And here is what happens when I grab the bottom edge of the window and drag it downwards:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_vertical.PNG" alt="Good vertical aspect ratio" width="600"/>
</p>

You can see that vertical black bars appeared in the first image, and horizontal black bars appeared in the second one (note that you can't see the top horizontal black bar in the second image because the sky is black in my scene).

To achieve those results my code did exactly what I described before: find the biggest area that fits inside the framebuffers and that has the original aspect ratio of `1.777`, and specify that area using `glViewport`.

Although my code did one more thing that I haven't mentioned: once it found the area, it centered it within the framebuffers. This last step is what results in the vertical and horizontal black bars:

- In the first image the area was centered horizontally, which is why vertical black bars appeared.
- In the second image the area was centered vertically, which is why horizontal black bars appeared.

One thing that I want to clarify is that I'm not making any calls to render those black bars. After the drawing area has been specified using `glViewport`, the call to `glClear(GL_COLOR_BUFFER_BIT)` that I make to clear the framebuffers in my render loop is still clearing the entire framebuffers, not just the drawing area. So in each iteration of the render loop the entire framebuffers become black, and then we draw the scene in the area specified by `glViewport`, leaving the space outside of that area black. That space that we never draw anything in is the black bars. If you change the color that's used to clear the framebuffers by calling `glClearColor`, you can change the color of the bars to anything you like.

So what does the code that implements this behaviour look like? We only need to make changes to the `framebufferSizeCallback` function. Let's start with a simple version of that function that calculates the correct area but doesn't center it:

{% highlight cpp %}
void framebufferSizeCallback(GLFWwindow* window,
                             int widthOfFramebuffer,
                             int heightOfFramebuffer)
{
    // This is the aspect ratio that we wish to maintain
    float desiredAspectRatio  = 1280.0f / 720.0f;

    // These are the two values that we will be calculating in this function
    int widthOfViewport, heightOfViewport;

    // Let's say that we want to use the width of the framebuffer as
    // the width of the viewport. What height would the viewport need to maintain the
    // desired aspect ratio of 1.777? This is a simple rule of three calculation
    float requiredHeightOfViewport = widthOfFramebuffer * (1.0f / desiredAspectRatio);

    // If the height required to maintain the aspect ratio is greater than
    // the height of the framebuffer, then we cannot use it because our viewport
    // wouldn't fit inside the framebuffer
    if (requiredHeightOfViewport > heightOfFramebuffer)
    {
        // Since using the width of the framebuffer as the width of the viewport failed,
        // let's try using the height of the framebuffer as the height of the viewport.
        // Now the question is: what width would the viewport need to maintain the desired
        // aspect ratio? Just as before, this is a simple rule of three calculation
        float requiredWidthOfViewport = heightOfFramebuffer * desiredAspectRatio;

        // If the width required to maintain the aspect ratio is greater than
        // the width of the framebuffer, then we cannot use it because our viewport
        // wouldn't fit inside the framebuffer
        if (requiredWidthOfViewport > widthOfFramebuffer)
        {
            // If we reach this point, we failed to find a width/height combination that
            // maximizes the area and preserves the aspect ratio.
            // This should never happen, though. It's always possible to find a
            // good combination. If we reach this point, then something is wrong in our code
            std::cout << "Error: Couldn't find dimensions that preserve the aspect ratio\n";
        }
        else
        {
            // Using the height of the framebuffer as the height of the viewport allowed
            // us to find a width for the viewport that maintains the aspect ratio
            // When this happens, you will observe vertical bars
            widthOfViewport = static_cast<int>(requiredWidthOfViewport);
            heightOfViewport = heightOfFramebuffer;
        }
    }
    else
    {
        // Using the width of the framebuffer as the width of the viewport allowed
        // us to find a height for the viewport that maintains the aspect ratio
        // When this happens, you will observe horizontal bars
        widthOfViewport = widthOfFramebuffer;
        heightOfViewport = static_cast<int>(requiredHeightOfViewport);
    }

    // Call glViewport to specify the new drawing area
    glViewport(0, 0, widthOfViewport, heightOfViewport);
}
{% endhighlight %}

With those changes in place, here is what happens when I grab the right edge of the window and drag it towards the right:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_horizontal_not_centered.PNG" alt="Good horizontal aspect ratio" width="728"/>
</p>

And here is what happens when I grab the bottom edge of the window and drag it downwards:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_vertical_not_centered.PNG" alt="Good vertical aspect ratio" width="600"/>
</p>

You can see that the aspect ratio is being maintained, but the drawing area isn't being centered. That's why there's only one black bar on the right in the first image, and one black bar on the top in the second image (although you can't see that one because the sky is black in my scene).

Fixing this is really simple. Below is the same code as before but with the necessary changes to center the viewport:

{% highlight cpp %}
void framebufferSizeCallback(GLFWwindow* window,
                             int widthOfFramebuffer,
                             int heightOfFramebuffer)
{
    float desiredAspectRatio  = 1280.0f / 720.0f;

    int widthOfViewport, heightOfViewport;
    // These are two new values that we will be calculating in this function
    int lowerLeftCornerOfViewportX, lowerLeftCornerOfViewportY;

    float requiredHeightOfViewport = widthOfFramebuffer * (1.0f / desiredAspectRatio);
    if (requiredHeightOfViewport > heightOfFramebuffer)
    {
        float requiredWidthOfViewport = heightOfFramebuffer * desiredAspectRatio;
        if (requiredWidthOfViewport > widthOfFramebuffer)
        {
            std::cout << "Error: Couldn't find dimensions that preserve the aspect ratio\n";
        }
        else
        {
            // Remember that if we reach this point you will observe vertical bars
            // on the left and right
            widthOfViewport = static_cast<int>(requiredWidthOfViewport);
            heightOfViewport = heightOfFramebuffer;

            // The widths of the two vertical bars added together are equal to the
            // difference between the width of the framebuffer and the width of the viewport
            float widthOfTheTwoVerticalBars = widthOfFramebuffer - widthOfViewport;

            // Set the X position of the lower left corner of the viewport equal to the
            // width of one of the vertical bars. By doing this, we center the viewport
            // horizontally and we make vertical bars appear on the left and right
            lowerLeftCornerOfViewportX = static_cast<int>(widthOfTheTwoVerticalBars / 2.0f);
            // We don't need to center the viewport vertically because we are using the
            // height of the framebuffer as the height of the viewport
            lowerLeftCornerOfViewportY = 0;
        }
    }
    else
    {
        // Remember that if we reach this point you will observe horizontal bars
        // on the top and bottom
        widthOfViewport = widthOfFramebuffer;
        heightOfViewport = static_cast<int>(requiredHeightOfViewport);

        // The heights of the two horizontal bars added together are equal to the difference
        // between the height of the framebuffer and the height of the viewport
        float heightOfTheTwoHorizontalBars = heightOfFramebuffer - heightOfViewport;

        // We don't need to center the viewport horizontally because we are using the
        // width of the framebuffer as the width of the viewport
        lowerLeftCornerOfViewportX = 0;
        // Set the Y position of the lower left corner of the viewport equal to the
        // height of one of the vertical bars. By doing this, we center the viewport
        // vertically and we make horizontal bars appear on the top and bottom
        lowerLeftCornerOfViewportY = static_cast<int>(heightOfTheTwoHorizontalBars / 2.0f);
    }

    // Call glViewport to specify the new drawing area
    // By specifying its lower left corner, we center it
    glViewport(lowerLeftCornerOfViewportX, lowerLeftCornerOfViewportY,
               widthOfViewport, heightOfViewport);
}
{% endhighlight %}

And that's it! With the code above you will never see any unpleasant stretching again.

As a bonus, I wanted to discuss one last problem: let's say that instead of having a black sky in my scene, I wanted it to be blue. I can implement that change by calling `glClearColor` with the RGB values of a nice shade of blue, like for example `glClearColor(0.036f, 0.627f, 1.0f, 1.0f)`. Now when my framebuffers are cleared in each iteration of my render loop, they become blue instead of black. This works perfectly, but notice what happens when the window is resized horizontally:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_all_blue.PNG" alt="Good horizontal aspect ratio with blue bars" width="728"/>
</p>

The bars on the sides look as if they were part of the sky, which isn't very nice. It would be better if they were black so that it was clear that they are not part of the scene, but how do we achieve that? It's actually really easy! First you need to add the following call below the call to `glViewport` in the `framebufferSizeCallback` function:

{% highlight cpp %}
glScissor(lowerLeftCornerOfViewportX, lowerLeftCornerOfViewportY,
          widthOfViewport, heightOfViewport);
{% endhighlight %}

And then in your render loop, in the place where you do this:

{% highlight cpp %}
// Set the clear color to the blue of the sky
glClearColor(0.036f, 0.627f, 1.0f, 1.0f)
// Clear the current framebuffer
glClear(GL_COLOR_BUFFER_BIT);
{% endhighlight %}

You need to do this instead:

{% highlight cpp %}
// Set the clear color to black
glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
// Clear the current framebuffer
// At this point the entire framebuffer is black
glClear(GL_COLOR_BUFFER_BIT);

// Set the clear color to the blue of the sky
glClearColor(0.036f, 0.627f, 1.0f, 1.0f);
// Clear the area within the scissor box, that is, the viewport
glEnable(GL_SCISSOR_TEST);
glClear(GL_COLOR_BUFFER_BIT);
glDisable(GL_SCISSOR_TEST);
{% endhighlight %}

With those changes in place, our application now looks quite cinematic:

<p align="center">
<img src="/assets/images/constant_aspect_ratio/good_ar_blue_and_black.PNG" alt="Good horizontal aspect ratio with black bars" width="728"/>
</p>

If you would like to see the code that I described here in action, open [this](https://diegomacario.github.io/Animation-Experiments) link and resize your browser's window. You will see that the aspect ratio never changes.
