+++
title = "CSE 167 Final Project - Results!"

[taxonomies]
tags = ["3d", "gamedev"]

[extra]
location = "San Diego"
+++

After a week of hard work and dedication my partner and I finally completed
our final project for our Computer Graphics class. Below is the description
we sent in to our professor as part of our project describing what we did.

<!-- more -->

> Picture a lone survivor of a dangerous war that has ripped his world apart. Our hero, a penguin,
> the last of a mighty and powerful race hurtles through space seeking shelter, refuge, and some cheesy snacks.
>
> Our final project scene consists of a procedurally generated planet that sits against a starry night sky.
> The planet is separated into 4 levels, ocean, coast, lush terrain, and mountains.
> The planet's lush terrain zone is spotted with fractal trees randomly dispersed across the planet.
> All of this is done in toon shading, modified slightly so that the world looks as if it was sketched.
> Zooming down from space following a bezier path, you'll fly past the planet's bump-mapped moon and
> once landed on the planet you'll be able to navigate the planet as a penguin using the keyboard
> W,A,S,D keys and the mouse. A scene graph is used to handle rendering and animation of the planet,
> trees and penguin. To handle drawing the huge amount of trees on the surface of the planet
> we use a combination of view frustum culling and a basic form of occlusion culling.
>
> The key 'T' can be used to toggle textures, 'O' toggles outlines, 'P' toggles the planet rendering,
> 'G' toggles toon shader, 'N' toggles planet normals, '<' zooms out of the planet, '>'
> zooms towards the planet, '/' stops the zoom action, 'B' toggles bump mapping of the moon,
> 'C' toggles the ability to see the bezier curve path, 'R' randomly generates another planet.

Good stuff right? As part of a series of articles I plan on spending my
winter break writing on how we managed to do each part, this will be the
zeroth! The prelude if you may where I show off all the cool stuff we did
to whet your appetite for more! A little warning, at lot of this will be a
rehash of what I said in previous posts since this is the final summary of
everything we did.

![](/img/CSE167/8612367-0-hurtling.png.scaled.500.jpg)

The very beginning of our scene starts off with us looking directly at a
small planet and it's moon. From there we zoom in on the planet whipping
around the moon in the process to show off the bump mapping we applied to
it. [Bump mapping](http://en.wikipedia.org/wiki/Bump_mapping) is a great
technique in computer graphics applications, that simply put, is used to
make a simple object (such as a sphere in our instance) look more detailed
then it really is. In our case the moon looks as if has ridges and craters,
and ergo much more highly detailed than a simple smooth sphere.

![](/img/CSE167/8612367-0-Screen_shot_2009-12-11_at_2.53.png.scaled.500.jpg)

Moving along in our scene we approach our planet in with the ability to see
the finer details of the terrain. The terrain itself was procedurally
created using a slightly modified version of [particle
deposition](http://www.lighthouse3d.com/opengl/terrain/index.php3?particle),
a really nice terrain generation algorithm that gives us that nice
archipelago look I was trying to achieve. The trees are also procedurally
generated using a simple fractal algorithm and are randomly dispersed on
the planet's "green" zone. Now the problem with rendering around 500
fractal trees on screen that the same time is that it's incredibly
time-consuming and our frame rate took a huge hit when it was first
implemented. To deal with this we use a simple form of occlusion culling,
which means that anything behind the planet will simply not be drawn. One
of the screenshots attached shows how trees behind the planet are not being
rendered at all. This speeds up rendering of the scene by around 2x-3x
based on how many trees are cut out of the rendering pipeline.

![](/img/CSE167/8612367-0-Screen_shot_2009-12-11_at_2.54.png.scaled.500.jpg)
![](/img/CSE167/8612367-0-0Screen_shot_2009-12-11_at_2.54.png.scaled.500.jpg)

And finally we land on the planet where we are able to move around as a
little penguin across the surface of the planet. Movement is a bit hacked
together at this point, and in all honestly sucks, but we were luckily able
to get together a semi-working version done by the time presentations were
due. The penguin is a simple 3D model created in
[Blender](http://www.blender.org), and is rotated and moved along the
planet based on where it is on the planet.

![](/img/CSE167/8612367-0-Screen_shot_2009-12-11_at_2.56.png.scaled.500.jpg)

All in all the presentation went well, people were amazed at the level of
complexity we managed to achieve in a relatively short time and we had tons
of fun doing it! If you're interested in checking out projects by other
groups, feel free to check out:
[http://graphics.ucsd.edu/twiki/bin/view.pl/Classes/CSE167F09-FinalProjects](http://graphics.ucsd.edu/twiki/bin/view.pl/Classes/CSE167F09-FinalProjects)
