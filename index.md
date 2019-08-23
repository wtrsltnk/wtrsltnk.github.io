---
title: wtrsltnk
author: Wouter Saaltink
layout: default
---
{% for post in site.posts %}
<div class="post col-lg-12">
    <h4 class="list-group-item-heading"><a href="{{ post.url }}">{{ post.title }}</a></h4>
    <p class="list-group-item-text">{{ post.excerpt }}</p>
    <div class="btn-group float-right">
        <a class="btn btn-link" href="{{ post.url }}">Read more</a>
    </div>
    <div class="clearfix"></div>
</div>
{% endfor %}    
