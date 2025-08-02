+++
title = 'The PlayStation Texture Wobble'
date = '2025-08-02'
draft = true
toc = true
tocBorder = true
math = true
[params]
  author = "Jordan Emme"
+++


## What's this about?

I just wanted to write a blog post about something funny I have learned relatively recently. If you
have played any PlayStation 1 games, you'll know that the graphics --- the textures in particular
--- seem to always contort and distort in strange ways as the viewing angles of the objects change.
For illustrative purposes (and for fun), I quickly slapped together a tiny software renderer I wrote
which somewhat emulates this behaviour on a rotating cube. Here is the pixelated output:

{{< video
  src="psone-cube.mp4"
  type="video/mp4"
  preload="auto"
>}}

Notice in particular the jarring distortions along a diagonal of each square face of this cube (this
happens to be the diagonal along which each square is partitioned into two triangles). Now I am not
pretending in any way that what I hacked is in any way, shape, or form a faithful representation of
how the PlayStation does all its rendering, but the fundamental reason why my renderer exhibits this
warp/wobble of the texture during the cube rotation is the same as why this phenomenon appeared on
the console.

Should you want to see a more faithful example of PlayStation graphics, do yourself a favour, and
watch a walkthrough of [some classic](https://www.youtube.com/watch?v=sbKBbTqTeT8), or better yet,
go and play some (if you can still get past the particular kind of jank that usually came with games
of that particular era that is).

## How Did the PlayStation One Compute Texture Coordinates?

To the best of my understanding, the factor which most contributes to the texture wobble effect in
PlayStation 1 games is the way the texture coordinates are computed. If you know the basics of how
to rasterise a triangle, and what UV/texture coordinates are in the context of 3D meshes, feel free
to jump to subsection [Affine Interpolation](#affine-interpolation).

### Basics of Triangle Rasterisation

First a little refresher (or crash-course) on how 3D is being renderer on screen through
rasterisation (there are other ways to render 3D objects on a 2D display, notably ray-tracing, but
we are specifically talking about rasterisation here). I'll only explain what is strictly
necessary for this post to be self containted, but should you want some more details,
or even write your own first renderer, [here is a great first introduction to the
matter](https://github.com/ssloy/tinyrenderer).

What is rasterisation? Put simply, it is the act of representing a triangle in 3D space as a bunch
of pixels on a 2D display. Let us precise the concepts with some C code (with no regard for performance: it rather tries to organise data in a didactic way).

```c
typedef struct vec3 {
  float x;
  float y;
  float z;
} vec3;

typedef struct tri3D {
  vec3 a;
  vec3 b;
  vec3 c;  
} tri3D;
```

We simply define a triangle in 3D space `tri3D` as three `vec3`s (or `float3` or whatever the hell
you want to name a vector in $\mathbb{R}^3$).

Of course, our displays are not 3D, but 2D, so we have to project these onto a 2D space. Let us
keep things simple and not worry too much about the actual "physical" properties of our display
(resolution, curvature if you have a CRT or a Gamerz™ ultrawide) for now, and consider a plane in
the 3D space which will represent the display. We'll call it the screen. There are three main steps:
1. Project the triangle to the screen.
2. Compute a bounding box for the projected 2D triangle.
3. Determine which pixels (with the appropriate discretisation of the plane) in the bounding box are
inside the projected triangle: these are the ones to be drawn.

#### Projection

There are many ways to project the three-dimensional space $\mathbb{R}^3$ to the two-dimensional
space $\mathbb{R}^2$. Of these, the two most notable ones in the domain of computer graphics are the
orthogonal projection and the perspective projection. The careful reader will have noted that I am
implying unicity of these projections through the use of the definite article: this is because it is
traditional to fix the screen to be the plane with normal $\vec{e_z} = (0,0,1)$ and containing the
point $O = (0,0,\pm 1)$ (the sign depending on some choice of conventions for coordinate systems) as
expressed in the observer's/camera's coordinate system.

In this context, the orthogonal projection is simply the function:
$$
\begin{align*}
p:\mathbb{R}^3 &\rightarrow \mathbb{R}^3 \\\
     (x, y, z)&\mapsto     (x, y , 1) 
\end{align*}
$$

This can be a very useful in certain cases, in particular in 2D games where you still want some
depth information between your sprite but don't want that to be reflected in any perspective in the
rendering. For a 3D game to provide a sense of depth and distances however, this is not the way to
go. We instead use a perspective projection as (brilliantly) illustrated below.

![projection](projection.png)

In this instance, the projection is simply:
$$
\begin{align*}
p:\mathbb{R}^3\setminus \\{z = 0\\} &\rightarrow \mathbb{R}^3 \\\
     (x, y, z)&\mapsto     (\frac{x}{z}, \frac{y}{z} , 1) 
\end{align*}
$$

Note the constraint here of needing to have $z \neq 0$. This is fine for our purpose, we would not
want to render things too close to the camera anyways, so if $z$ is too small we can just chuck it
away (well, sort of...).

For reasons that will become clear later, we should also store the value $\frac{1}{z}$, sometimes
denoted $w$ (for weight), so one could compute the projected coordinates in the following way.

```c
vec3 proj(vec3 v) {
  float w = 1.f / v.z;
  return (vec3) {.x = v.x * w, .y = v.y * w, .z = w};
}  
```

#### Bounding Box



#### Determining if a Point is in a Triangle

### How Texturing Works

### Affine Interpolation

## What's Wrong with Affine Texture Interpolation?


