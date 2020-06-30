+++
title = "Trees and Penguins, Oh My!"

[taxonomies]
tags = ["code"]

[extra]
location = "San Diego"
+++

Update #2 on our CSE 167 Final Project

<!-- more -->

## Trees

![Trees!](/img/CSE167/8488720-0-Screen_shot_2009-12-09_at_12.3.png.scaled.500.jpg)

Previously I said we'd try and populate the planet with trees, and as you can see in one the screenshots this has been completed! The trees themselves are fractal trees and procedurally created ( they all look the same however, but a simple rand() call to modify branch position, etc should fix that ). The tree positions are calculated based on the planet's height map so that trees don't end up where they shouldn't be ( such as in the ocean or on top of a mountain ) and are randomly distributed across the planet's "green zone". Looks quite nifty doesn't it =].

## Object Culling

![Hiding objects](/img/CSE167/8488720-0-0Screen_shot_2009-12-09_at_12.3.png.scaled.500.jpg)

All in all, we have around 500 trees on screen at a time which is highly detrimental to our frame rate. To remedy this, we have two forms of culling in action at the moment. One is the normal view frustum culling, which means we simply do not render any objects that can not be seen in the view of the camera, and second is a basic form of occlusion culling. Now occlusion culling essentially means that we don't draw any objects that are being blocked by another object, which works well with our planet and trees scene. If you look at the screenshots I attached, you can see that on one of them, with the planet rendering turned off, you can see only half the trees are actually being rendered. This improves our frame rate by around 2x depending on how sparse the trees are on either side of the planet.

## Penguins

![Penguins!](/img/CSE167/8488720-0-Screen_shot_2009-12-09_at_12.1.png.scaled.500.jpg)

And finally our protagonist in the scene, our beloved penguin. The penguin is a model I created when learning [Blender](http://www.blender.org), an open source 3D modeling and animation program. As of now all we can do is render the penguin on the planet. Moving it along with the camera with a little animation added is in the works, but should hopefully be done by tonight! Hope you enjoyed the update, I should hopefully post the final results of our project sometime in the wee hours of the morning tomorrow =P.