---
layout: post
title: "Welcome to Jekyll!"
tags: C++ OpenGL
---

You’ll find this post in your {% ihighlight cpp %}float squaredLen = q.x * q.x + q.y * q.y + q.z * q.z + q.w * q.w;{% endihighlight %}  directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight cpp %}
Q::quat Q::normalized(const Q::quat& q)
{
   float squaredLen = q.x * q.x + q.y * q.y + q.z * q.z + q.w * q.w;
   if (squaredLen < QUAT_EPSILON)
   {
      return Q::quat();
   }
   float invertedLen = 1.0f / glm::sqrt(squaredLen);

   return Q::quat(q.x * invertedLen,
                  q.y * invertedLen,
                  q.z * invertedLen,
                  q.w * invertedLen);
}
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
