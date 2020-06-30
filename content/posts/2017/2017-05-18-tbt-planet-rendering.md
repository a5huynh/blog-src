+++
title = "TBT: Planet Rendering"

[taxonomies]
tags = ["gamedev", "3d"]

[extra]
location = "San Francisco"
+++

Many moons ago, I worked on a college project that required procedurally
generating a "realistic" planet in 3D space. The original implementation created
a mesh in the form of the [UV sphere](https://en.wikipedia.org/wiki/UV_mapping)
and then applied a [particle deposition][pd-link] to create the terrain.

<!-- more -->

I thought it would be a fun project to revisit my implementation and use modern
tooling to re-create what I had done close to 10 years ago in vanilla javascript
and WebGL with [Typescript](https://www.typescriptlang.org) and
[Three.js](https://threejs.org/).

> You can view the old project and newer code used in this post
> [here](https://github.com/a5huynh/planet-generator).


## Rendering the sphere

The UV sphere was originally chosen due to time constraints. The code (shown
below) to generate a UV sphere is relatively compact and easy to follow
making for a quick implementation and allowing us to focus on the terrain
generation and animation used in the project.

``` javascript
    // Columns
    var gamma = 0.0;
    var gammaStep = 2 * Math.PI / numFaces;

    // Rows
    var theta = 0.0;
    var thetaStep = Math.PI / numFaces;

    // Loop through each row & col in spherical coordinates.
    //
    // Note: numPoints = numFaces + 1. You need n + 1 points to
    // represent n faces.
    for(var row = 0; row < numPoints; row++) {
        gamma = row * gammaStep;
        for(var col = 0; col < numPoints; col++) {
            theta = col * thetaStep;

            // Convert to cartesian coordinates
            var x = Math.sin(theta) * Math.cos(gamma);
            var y = Math.cos(theta);
            var z = Math.sin(theta) * Math.sin(gamma);

            var v = new Vector3(x, y, z).normalize();
            this.vertices.push(v);
        }
    }
```
*Using latitudinal and longitudinal lines as a grid, we can loop through each
row/column and generate points for the sphere.*

Combining the above code with some Three.js materials and lighting, we get the
following scene below, a blank canvas in which we can start transforming the
surface into believable terrain. Unfortunately at the time, we didn't notice how
the area of the faces changed as you moved closer to the equator and poles of
the sphere. As you'll see later on, this will come back to bite us when we start
to apply the tranformations.

<iframe
    scrolling="no"
    class="full-width"
    src="/webgl/planet-generator/index.html?sphereType=uv&zoom=3"
></iframe>

Now onto the terrain generation! How do we take this flat spherical surface and turn it into
the rugged planet we want for our scene?


## Particle Deposition

Particle deposition starts with a single point and slowly builds out a land mass
around that area by randomly moving different directions adding additional
height displacement as it goes. If we'd like, at the end of the terrain
generation we can then apply a smoothing function on the resulting land mass to
make it look a little less jagged.

``` typescript
    for( var i = 0; i < numIslands; i++ ) {
        // Find a random spot on the planet to grow an island
        var sx = randomInt( 0, width );
        var sy = randomInt( 0, height );

        for( var itNum = 0; itNum < iterations; itNum++ ) {
            // Add the displacement to the current height
            set( sx, sy, get( sx, sy ) + displacement );

            // Pick a direction to go next
            switch( this._randomInt( 0, 3 ) ) {
                case NORTH: sx = sx + 1; break;
                case EAST:  sy = sy + 1; break;
                case SOUTH: sx = sx - 1; break;
                case WEST:  sy = sy - 1; break;
            }
        }
    }
```
*Note that it's possible to continue moving along a single direction,
generating a ridge. We can box in land masses by making sure future directions are
with `n` of the initial growth point.*

To see the results of the above code, lets start off with a single landmass and
apply particle deposition. In the following example, I've bumped up the
resolution of the sphere so that a single island is more noticeable.

<iframe
    scrolling="no"
    class="full-width"
    src="/webgl/planet-generator/index.html?sphereType=uv&amp;terrainType=particle&amp;planetDetail=50&amp;zoom=3&amp;islands=1"
></iframe>

### Trouble in Paradise

As we continued with the terrain generation, one of the main issues we faced was
due to how a UV sphere is represented in 3D space. As mentioned previously, the
sphere is divided up into meridians and parallels, which creates an effect
where faces towards the top and bottom of the sphere are smaller and more
bunched together, while the faces that are closer to the middle of sphere are
larger and more spread out.

> Fun fact: This sort problem also exists in how we represent Earth. Different
> [map projections](https://en.wikipedia.org/wiki/Map_projection) will show
> landmasses in different ways and may distort the actual area/size/shape.

This distortion can be seen in full effect by bumping up the number of land
masses. It wasn't ideal, but since we were only using the planet as part of an
animation, as long as we focused on the mid-section of the sphere the distortion
isn't as evident.

<iframe
    scrolling="no"
    class="full-width"
    src="/webgl/planet-generator/index.html?sphereType=uv&amp;terrainType=particle&amp;planetDetail=50&amp;zoom=3&amp;particleNumIslands=20">
</iframe>

## For Next Time

In the next post, we'll look at how we can apply height based vertex coloring
to give our planet more personality as well as other sphere rendering methods
that lead to a more natural looking planetoid.

Hope you enjoyed this short foray into sphere rendering and terrain generation.


> You can view the old project and newer code used in this post
> [here](https://github.com/a5huynh/planet-generator).

## Edits

* 2018-04-13: Updated link to particle deposition tutorial.

[pd-link]: http://www.lighthouse3d.com/opengl/terrain/index.php?particle