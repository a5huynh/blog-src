+++
title = "Planet Rendering Pt. 2"

[extra]
location = "Tokyo/San Francisco"
+++

A long-awaited continuation of my [previous post](/2017/tbt-planet-rendering),
in this one we will go into the exciting intricacies of:

* Applying vertex coloring to give our planet a little more personality,
* Rendering a new type of sphere, the icosahedron, and
* Projecting a terrain map onto the icosahedron.

<!-- more -->

> You can view the code used in this post
> [here](https://github.com/a5huynh/planet-generator).


## Vertex Coloring

Adding a little color to our planet will turn it from the boring blue orb
we've seen previously to the more realistic, colorful planet seen below!

<iframe scrolling="no"
        class="full-width"
        src="/webgl/planet-generator-v2/index.html?sphereType=uv
&amp;terrainType=particle
&amp;planetDetail=50
&amp;zoom=3
&amp;particleNumIslands=10">
</iframe>

To accomplish this task, we first divide the terrain height map into
different regions and assign a color based on the height of that particular
vertex as we create the sphere. This only requires a tiny refactoring of
our current code due to how Three.js handles vertex colors. Choosing
offsets is a bit of trial and error depending on how large we choose the
displacement factor in terrain generation. In this particular case, since
our displacement factor is pretty tiny (0.1), our offsets reflect that.

``` typescript
_color( height: number ): THREE.Color {
    // Ocean
    if( height <= 1.01 ) {
        return new THREE.Color(0x55bbff);
    // Sand
    } else if( height <= 1.05 ) {
        return new THREE.Color(0xffff00);
    // Grassland
    } else if( height <= 1.15 ) {
        return new THREE.Color(0x00ff00);
    // Mountains
    } else {
        return new THREE.Color(0xfffffff);
    }
}
```
*Coloring is as simple as finding a cutoff for the terrain type we're trying
to represent. In this case, we're splitting the terrain into ocean, sand,
grassland, and mountains.*

Secondly, we modify how we create `Face3` objects, using the `_color`
function above to assign a color upon creation.

``` typescript
_face( x: number, y: number, z: number ): THREE.Face3 {
    let face = new THREE.Face3( x, y, z );
    face.vertexColors[0] = this._color(
        this.geometry.vertices[face.a].length()
    );
    face.vertexColors[1] = this._color(
        this.geometry.vertices[face.a].length()
    );
    face.vertexColors[2] = this._color(
        this.geometry.vertices[face.a].length()
    );
    return face;
}
```

Now that we have the vertex colorization accomplished, let's take a stab at
fixing the distortion in the UV sphere geometry through the use of another
sphere geometry -- the icosahedron.

## Icosahedron

The [icosahedron](https://en.wikipedia.org/wiki/Icosahedron) is a
polyhedron with 20 faces where each face retains its form no matter where
it appears on the sphere, unlike the UV sphere.

Due this handy property, we can use the icosahedron as a starting point for
a more realistic looking planetoid. Through subdividing the faces in
increasing levels of detail, the icosahedron becomes a smooth,
non-distorted sphere as seen below.

<iframe scrolling="no"
        class="full-width"
        src="/webgl/planet-generator-v2/index.html?sphereType=iso
&amp;sphereDetail=20
&amp;zoom=3">
</iframe>

The algorithm to generate the sphere is pretty straightforward. The
icosahedron starts off with 20 faces created using a set of 12 initial
vertices.

``` typescript
_setupInitialVertices() {
    let t = ( 1 + Math.sqrt(5.0)) / 2.0;

    this.geometry.vertices.push(
        this._normalizedVector( -1,  t,  0 ),
        this._normalizedVector(  1,  t,  0 ),
        this._normalizedVector( -1, -t,  0 ),
        this._normalizedVector(  1, -t,  0 ),
    );

    this.geometry.vertices.push(
        this._normalizedVector(  0, -1,  t ),
        this._normalizedVector(  0,  1,  t ),
        this._normalizedVector(  0, -1, -t ),
        this._normalizedVector(  0,  1, -t ),
    );

    this.geometry.vertices.push(
        this._normalizedVector(  t,  0, -1 ),
        this._normalizedVector(  t,  0,  1 ),
        this._normalizedVector( -t,  0, -1 ),
        this._normalizedVector( -t,  0,  1 )
    );
}
```
*All we need to generate the initial vertices is a single calculation of the
golden ratio (\(\tau\)) and then applying that around the sphere.*

Afterwards, we'll setup the initial faces and based on the level of detail
we want, divide the initial faces into smaller and smaller triangles.

``` typescript
setupInitialFaces( ) {
    // 5 faces around point 0
    this.faces.push( this._face(  0, 11,  5 ) );
    this.faces.push( this._face(  0,  5,  1 ) );
    this.faces.push( this._face(  0,  1,  7 ) );
    this.faces.push( this._face(  0,  7, 10 ) );
    this.faces.push( this._face(  0, 10, 11 ) );

    // ... see code for the rest of the face setup code.
}

// Refine the icosahedron geometry based on the
// level of detail we want.
refineGeometry() {
    for( var i = 0; i < level_of_detail; i++ ) {
        let refinedFaces = new Array<THREE.Face3>();
        for( let triangle of this.faces ) {
            // Replace the triangle with 4 new triangles.
            let a = getMidPoint( triangle.a, triangle.b );
            let b = getMidPoint( triangle.b, triangle.c );
            let c = getMidPoint( triangle.c, triangle.a );

            refinedFaces.push(
                this._face( triangle.a, a, c ),
                this._face( triangle.b, b, a ),
                this._face( triangle.c, c, b ),
                this._face( a, b, c ),
            )
        }
        this.faces = refinedFaces;
    }
}
```

And voila! One complication with the icosahedron geometry is the mapping of
the generated terrain heights to the vertices. Since the icosahedron
geometry is not a grid like the UV sphere, we'll have to estimate where a
particular vertex would be in the grid.

Ideally, we'd update our terrain generation code to work nicely with the
icosahedron geometry, but this saves us a little bit of time and makes it
so we can reuse the same terrain generation code with different geometries.


## 2D terrain map on a 3D surface

Projecting a 2D terrain map onto the 3D surface will require a little
trigonometry. As we're creating the vertices for the planet, we'll convert
the `(x, y, z)` coordinate into a `(x, y)` coordinate that maps onto the
terrain. This technique is actually very similar to how you would generate
texture coordinates to apply a texture to the icosahedron geometry.

Lets take a look at the updated `getHeight` function in the icosahedron planet
generator below:

``` typescript
getHeight( x: number, y: number, z:number ): number {
    var hx = Math.atan2( x, z ) / ( -2.0 * Math.PI );
    var hy = Math.asin( y ) / Math.PI + 0.5;

    if ( hx < 0 ) {
        hx += 0.5;
    }

    // Since texture coordinates are between 0 and 1
    // we multiply by the terrain size to give us
    // a grid location within the terrain map.
    hx = Math.floor( hx * (this.terrain_size - 1) );
    hy = Math.floor( hy * (this.terrain_size - 1) );

    // Handle wrapping correctly.
    if( isNaN( hx ) ) {
        hx = this.terrain_size - 1;
    }

    if( isNaN( hy ) ) {
        hy = this.terrain_size - 1;
    }

    return this.terrain.getHeight( hx, hy );
}
```
*Here we use the inverse trigonometry functions `atan2` and `asin` to
convert the vertex coordinates into the `x, y` coordinates we can use to
reference a height map.*

Combined with the vertex coloring functionality developed earlier, we now
have the following, more colorful looking planetoid.

<iframe scrolling="no"
        class="full-width"
        src="/webgl/planet-generator-v2/index.html?sphereType=iso
&amp;terrainType=particle
&amp;planetDetail=16
&amp;zoom=3
&amp;particleNumIslands=15">
</iframe>

Further tweaking of the terrain generation and blending of the vertex
colors would fix some of the artifacts you'll probably see in the planet
below, such as the water/grass terrain stretching up into the mountains and
hard cutoffs rather than smooth transitions between terrain types.


> You can view the code used in this post
> [here](https://github.com/a5huynh/planet-generator).
