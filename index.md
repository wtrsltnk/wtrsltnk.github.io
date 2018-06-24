---
title: wtrsltnk
author: Wouter Saaltink
layout: default
---
<div class="jumbotron jumbotron-fluid">
  <div class="container">
    <h1 class="display-4">Fluid jumbotron</h1>
    <p class="lead">This is a modified jumbotron that occupies the entire horizontal space of its parent.</p>
  <hr class="my-4">
    <p class="lead">This is a modified jumbotron that occupies the entire horizontal space of its parent.</p>
  <p class="lead">
    <a class="btn btn-primary btn-lg" href="#" role="button">Learn more</a>
  </p></div>
</div>  
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
