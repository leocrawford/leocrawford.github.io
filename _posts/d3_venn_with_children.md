---
layout: post
title:  "D3 Venn with Constrained Children"
# date:   2016-10-06 13:31:25 +0100
# categories: code, d3
---
I came across [d3 venn](https://github.com/benfred/venn.js/) and had the idea of displaying the categorisation of nodes by placing them in the appropriate circle or intersection. In this way the size of each venn would be defined by the number of items within it.

Whilst figuring out how to do this I came accross <https://github.com/christophe-g/d3-venn> which was close to what I wanted [visually](http://bl.ocks.org/christophe-g/b6c3135cc492e9352797), but I wasn't that keen on the circle packing (and wasted space) and also found what seemed to be a [bug](https://github.com/christophe-g/d3-venn/issues/3).

I decided to have a tinker and see if I could get it work how I wanted, and also as an opportunity to learn javascript and d3. Before I explain what I did (and how I did it) take a look at the result:

<iframe src="/assets/d3_venn_with_children/dynamic_mod.html" frameborder="0" width="100%" height="800"> </iframe>

As you'll see each child node is rendered in the colour of its category and is pulled strongly (almost constrained) into the correct circle or intersection. If it is dragged out of its rightful place then it turns black to indicate the problem. The number of nodes in each circle or intersection can be changed, but to get it working I did remove the smooth animation on these. Maybe something to fix in due course.

The children nodes are all laid out together, as part of the same force layout. They are each pulled towards the centre of the correct circle or intersection, as well as having gravity towards each other. There is also a force to stop them overlapping.

The trickiest bit has been figuring out the bounding area of the circles. The original version had circles represented by their whole area, without any intersecting area removed. E.g. circle A in set A,B,C was defined as the whole area of A, without the removal of the intersections AB and AC and ABC. Whilst it was fairly straight-forward to create a path that chose different arcs I remain unsure about all the edge cases, in particular circles that have "impinging" circles that extend wider than the midpoint of the circle.

Now for the code, hosted as a [gist](https://gist.github.com/leocrawford/28d8b469c576871c46edef48e1863ddd)

{% gist 28d8b469c576871c46edef48e1863ddd %}
