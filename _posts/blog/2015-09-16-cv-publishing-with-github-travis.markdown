---
layout: post          #important: don't change this
title: "Publishing a markdown CV with Github, Travis, Docker and Pandoc"
date: 2015-09-16 18:16:00
author: Ashley
categories:
- blog                #important: leave this here
- github
- travis
- docker
- pandoc
- ...
img: post01.jpg       #place image (850x450) with this name in /assets/img/blog/
thumb: thumb01.jpg    #place thumbnail (70x70) with this name in /assets/img/blog/thumbs/
---
About 6 months ago I started trying to work with simplified CV using [Markdown](daringfireball.net/projects/markdown/).

Now I've got a CV which is source-controlled, and is published in multiple formats (HTML, PDF, Word)
on commit.

I had to use this feature in anger today having spotted the word 'technoolgy' in a Word-rendered version.

What I haven't done (yet) is have it /pushed/ to any potential readers.

<!--more-->

When I started, I tried to use various markdown-to-PDF converters, none of which was satisfactory (e.g. markdown-pdf seemed ok, but actually just renders text into an image, wrapped as a PDF - CWjob won't upload it once it goes > 1Mb).

Side note - I'm not dogmatic about the use of Markdown - it's pretty much just sugar for a subset of HTML. Sadly the likes of Jekyll have made it a de facto standard because of the metadata/boilerplate. (I could hhappily rant on about XML/HTML and properly namespaced metadata which would reduce the need for tooling.)

This I discovered pandoc, and more importantly how to tame it with Docker. 

## The Process

{% ditaa %}
                                         publishing flow                                                        
    +---------------------------------------------------------------------------------------------------------->
                                                                                                                
       +--------------+        +--------------+              +---------------+         +-----------------+      
       |              |        |              |              |               ++        |                 |      
       |  local git   |        |  remote git  |              |  published    ||        |   github        |      
       |      CV      |        |      CV      |              |  formats      ||        |   pages         |      
       |              |        |              |              |               ||        |                 |      
       +--+-----+-----+        +------+--+----+              +--^-------------|        +----^------------+      
          |     |                     ^  |                    +---------------+             |                   
          |     |                     |  |                      |    |                      |                   
          |     |     committed by    |  |      built by        |    |    published by      |                   
          |     |   +---------------+ |  |    +-------------+   |    |   +--------------+   |                   
          |     |   |               | |  |    |             |   |    |   |              |   |                   
          |     +--^+               +-+  +---->  travis/    +---+    +--->  travis      +---+                   
          |         |    git        |         |  docker     |            |              |                       
          |         |               |         |  pandoc     |            |              |                       
          |         +---------------+         +-----+-------+            +------+-------+                       
          |                                         ^                           ^                               
          |                                         |                           |                               
          |  locally built +----------------+       |     +-------------+       |                               
          +---------------->  docker/       |       |     |  travis     |       |                               
                           |  pandoc        |       +-----+  config     +-------+                               
          +----------------+                |             |             |                                       
          |                |                |             |             |                                       
          |                +----------------+             +-------------+                                       
          |                                                                                                     
          |                                                                                                     
       +--v------------+                                                                                        
       |  locally      +-+                                                                                      
       |  published    ||                                                                                       
       |  files        ||                                                                                       
       |               ||                                                                                       
       +----------------|                                                                                       
        +---------------+                                                                                       

{% endditaa %}

## Testing and Committing changes

Having edited the CV, I may want to test changes before committing.
This also tests the publishing process outside of travis: Testability is a big plus for docker.

There are several Docker images available - this one works for me...
    https://github.com/vpetersson/docker-pandoc

I will type 'make pdf' (or 'make all') to test locally. This just triggers the publishaing process within the pando container:
    docker run -v $(pwd):/docs vpetersson/pandoc /bin/sh -c pandoc -V geometry:margin=1in -t latex -o out/cv.pdf cv.md

It was almost surprising how easily pandoc handles HTML and Word publishing out of the box. There is scope to add styling (via tex/css) to suit taste. I don't feel the need to style my CV.
The need to publish to Word format is entirely down to agencies/HR stuck in the stone age, unable to field PDFs. Yes. In 2015.

## Publishing

There are numerious tutorials describing the use of Travis with Github pages.
I found [this Steve Klabnik post](https://github.com/steveklabnik/automatically_update_github_pages_with_travis_example) to be most enlightening, not least because it's published by the same method.

The magic is in his "deploy.sh" script.

Basically, Github will automatically publish any project branch named 'gh-pages' at github.io.  The publishing process builds a branch which contains only the generated files, and a CNAME file to handle requests from the www.bellesoft.uk domain.  The files are now accessible from the root http://www.bellesoft.uk/ash-cv/

