# 对不起，你是个好人，可惜最好的，都是死人



**博客首页**
{% for post in  site.posts %}
[{{ post.title }}]({{ post.url | prepend: site.baseurl }})
{% endfor %}
