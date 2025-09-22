+++
title = 'The PlayStation Texture Wobble'
date = '2025-09-21'
draft = false
toc = true
tocBorder = true
math = true
[params]
  author = "Jordan Emme"
+++

Where I ramble on about the PlayStation texture affine interpolation and a few bits on
how to render a triangle in software. [Some hacky code cobbled together for illustration
purposes](https://github.com/JordanEmme/s0-wren/).

## What's this About?

I just wanted to write a blog post about something funny I have learned relatively recently. If you
have played any PlayStation 1 games, you'll know that the graphics --- the textures in particular
--- seem to always contort and distort in strange ways as the viewing angles of the objects change.
For illustrative purposes, I quickly slapped together a tiny software renderer I wrote which
somewhat emulates this behaviour on a rotating cube. Here is the pixelated output:

{{< video
  src="psone-cube.mp4"
  type="video/mp4"
  preload="auto"
>}}

Notice in particular the jarring distortions along a diagonal of each square face of this cube (this
happens to be the diagonal along which each square is partitioned into two triangles). There are a
few factors that came into play to explain that phenomenon on the original hardware, and I'll
explain the main one here.

Should you want to see a more faithful example of PlayStation graphics, do yourself a favour, and
watch a walkthrough of [some classic](https://www.youtube.com/watch?v=sbKBbTqTeT8), or better yet,
go and play some (if you can get past the particular kind of jank that usually came with games of
that particular era that is).

## How Did the PlayStation Compute Texture Coordinates?

To the best of my understanding, the factor which most contributes to the texture wobble effect
in PlayStation 1 games is the way the texture coordinates are being computed. If you know
the basics of how to rasterise a triangle, and what texture coordinates are in the context
of 3D meshes, feel free to jump to subsection about [the PlayStation texture coordinates
computation.](#the-playstation-texture-coordinates-computation) Otherwise, please read on as I try
to make a simple, self-contained, summary of what is needed to understand the issue.

### Basics of Triangle Rasterisation

First, a refresher (or crash-course) on raterisation. I'll only explain what is strictly necessary
for this post to be self contained, but should you want some more details, or
even write your own first renderer, [here is a great first introduction to the
matter](https://github.com/ssloy/tinyrenderer).

What is rasterisation in our context? It is the act of representing a triangle (or any geometrical
object) in 3D space as a bunch of pixels on a 2D display. Let us make the concepts precise
with some C code (with no regard for performance: I rather try to organise code and data in a
didactic way).

```c
typedef struct vec3 {
  float x;
  float y;
  float z;
} vec3;

typedef struct tri3D {
  vec3 pos[3];
} tri3D;
```

We simply define a triangle in 3D space `tri3D` as three `vec3`s (or `float3` or whatever the hell
you want to name a vector in \\(\mathbb{R}^3).\\)

Displays being 2-dimensional, we have to project these onto a 2D space. Let us not worry about the
actual "physical" properties of our display (resolution, curvature if you have a CRT or a Gamerz™
ultrawide), and model the display by a plane in the 3D space. We'll call it the *screen*. There are
three main steps:

1. Project the triangle to the screen.
1. Compute a *bounding box* for the projected 2D triangle.
1. Determine which pixels in the bounding box are inside the projected triangle and shade them. 

---
{data-content=Projection}

There are many ways to project the three-dimensional *world space* \\(\mathbb{R}^3\\) to the
two-dimensional *screen space*. Of these, the two most notable ones in the domain of computer
graphics are the *orthogonal* projection and the *perspective* projection.

Let us fix some conventions about the orientation and basis of these spaces in order to formalise
these projections.

* The *world space* is the euclidian space \\(\mathbb{R}^3.\\)
* The *camera* (or *observer*) is placed at the origin \\(O.\\)
* The *screen space* is the plane of equation \\(z = 1.\\)
* The world coordinates are expressed in an orthonormal basis (also called *right-handed*).

> __Remark__: The orientation is dependent on some choice of coordinate systems as there is not a
single convention in the world of 3D modeling and computer graphics. Some software assume a direct
(also called right-hand) orientation --- which would be my preference, given this is the most
natural way to do things, mathematically speaking --- and some assume a left-hand orientation. My
code exhibits an unusual choice, with \\(x\\) pointing right,  \\(y\\) pointing down, and \\(z\\)
pointing forward. This makes the \\(xy\\) orientation consistent with the canonical display
orientation (inherited from scanlines era), the look position to be positive, and the overall
coordinate system to be direct.

In this context, the orthogonal projection is the function:
\\[
\begin{align*}
p:\mathbb{R}^3 &\rightarrow \mathbb{R}^3 \\\
     (x, y, z)&\mapsto     (x, y, 1) 
\end{align*}
\\]

This can be a very useful in certain cases, in particular in 2D games where you still want some
depth information between your sprite but don't want that to be reflected in any perspective in
the rendering.

For a 3D game to provide a sense of depth and distances however, we need to use a perspective
projection as illustrated below.

![projection](projection.png)

In this instance, the projection is:

\\[
\begin{align*}
p:\mathbb{R}^3\setminus P_z &\rightarrow \mathbb{R}^3 \\\
     (x, y, z)&\mapsto     \left(\frac{x}{z}, \frac{y}{z} , 1\right) 
\end{align*}
\\]

where \\(P_z\\) is the plane \\(\left\\{ (x, y, 0) \ | \ (x, y) \in \mathbb{R}^2 \right\\}.\\)

> __Remark__: Note that we need to impose \\(z \neq 0\\) in world space. This is fine, as we don't
want to render things too close to the camera anyways.

When coding this projection, we should also store the value \\(\frac{1}{z}\\), as we are going to
need it later. The \\(z\\) coordinate being always \\(1\\) after the projection, we can store it
there without needing additional data.

```c
vec3 proj(vec3 v) {
  float w = 1.f / v.z;
  return (vec3) {.x = v.x * w, .y = v.y * w, .z = w};
}  
```

---
{data-content="Bounding Box"}

Now we have some projected coordinates --- as well as the *weight* \\(w = \frac{1}{z}\\) --- we
need to figure out which pixels to shade. The idea is to bound the projected triangle in a *bounding
box* so as to only check the pixels in that box against the projected triangle. So we need this kind
of data:

```c
typedef struct BBox {
  uint16_t minRow;
  uint16_t maxRow;
  uint16_t minCol;
  uint16_t maxCol;
} BBox;
```

---
{data-content="Checking if a Point is in a Triangle"}

We want to check for every pixel in the bounding box whether they are in the projected triangle.
If it is in we shade it, else, we ignore it.

To check whether a point \\(P\\) in \\(\mathbb{R}^2\\) is in a non-degenerate triangle \\(ABC\\)
we need to express the coordinates of \\(P\\) as a weighed average of the coordinates of \\(A\\),
\\(B\\), and \\(C.\\)

More formally, say our plane has origin \\(O\\), then

\\[
\exists!\ (m_A, m_B, m_C) \in \mathbb{R}^3 \setminus \left\\{(0,0,0)\right\\},\quad \left\\{
\begin{align*}
&\overrightarrow{OP} =  m_A \overrightarrow{OA} + m_B \overrightarrow{OB} + m_C \overrightarrow{OC}
\\\
&\text{and} \\\
& m_A + m_B + m_C = 1.
\end{align*}
\right.
\\]

These three quantities \\(\left(m_A, m_B, m_C\right)\\) are the *barycentric coordinates* of \\(P\\)
in the plane defined by the triangle \\(ABC.\\) These coordinates satisfy a property which is of
interest to us:

\\[
P \in ABC \Leftrightarrow m_A \geq 0 \ \text{and} \ m_B \geq 0 \ \text{and} \ m_C \geq 0.
\\]

Thus we only need to compute these coordinates to be able to determine whether a point falls inside
or outside the triangle. There are many ways to compute them, but we'll just admit the following
formula, chosen by virtue of its symmetry and ease of computation:

> ##### Barycentric Coordinates Formula
> \\[
> \begin{aligned}
> m_A &= \frac{\text{SignedArea}(PBC)}{\text{SignedArea}(ABC)}, \\\
> m_B &= \frac{\text{SignedArea}(APC)}{\text{SignedArea}(ABC)}, \\\
> m_C &= \frac{\text{SignedArea}(ABP)}{\text{SignedArea}(ABC)}. 
> \end{aligned}
> \\]

Here, the [signed area](https://en.wikipedia.org/wiki/Signed_area) of a triangle \\(ABC\\) is the
determinant of a \\(3\times 3\\) matrix as follows:

> ##### Signed Area Formula
> \\[
> \text{SignedArea}(ABC) = \frac12 \left|
> \begin{array}{ccc}
> x_A & y_A & 1 \\\
> x_B & y_B & 1 \\\
> x_C & y_C & 1 
> \end{array}
> \right|.
> \\]

> __Remark__: Although only the signs of the barycentric coordinates are needed now, we will also
need the values later, so we might as well compute them at that earlier stage.

---
{data-content="A Quick Word on Depth"}

Lastly, we need to talk about *depth*, as it is both fundamental to rasterisation, and relevant to
the PlayStation's hardware shortcomings.

If your scene contains more than one triangle, it is plausible that one is in front of another
from the point of view of the camera. In that case, some pixel may need shading more than once, so
how can we choose which shade is the appropriate one?

Well, the appropriate pixel is the one from the triangle we can actually see, *i.e.* the one closest
to us: it is the one with the smallest (positive) *depth* \\(z\\)!

A standard method is to cache the depths of the pixels we have already encountered and check against
it for each pixel. If we already have a depth value at that pixel position, and it's smaller than
the current depth, we do nothing: there is something in front of it. Else, we draw it and update the
cache appropriately.

These days, GPUs have dedicated hardware for this cache, called the *z-buffer*, so game and engine
devs need not be concerned about it in the same way that Playstation era devs did, as they had to
program that buffer themselves, in software. Yes, for 3D games, on a console advertised as
3D-first. Bummer...

---
{data-content="Summary"}

Here is a crude code overview of the procedure thus far.

```c
for (size_t i = 0; i < numTriangles; ++i) {
  tri3D tri = triangles[i];
  tri3D projTri;
  BBox box = {NUM_ROW, 0, NUM_COL, 0};
  for (int j = 0; j < 3; ++ j) {
    vec3 vertPos = tri.pos[j];

    vec3 projPos;
    float w = 1.f / vertPos.z;
    projPos.x = w * vertPos.x;
    projPos.y = w * vertPos.y;
    projPos.z = w;

    projTri.pos[j] = projPos;
    update_box(&box, vertPos.x, vertPos.y);
  }

  for (int row = box.minRow; row <= box.maxRow, ++row) {
    for (int col = box.minCol; col <= box.maxCol, ++col) {
      if (pixel_is_in_tri(row, col, projTri)) {
        float depth = get_z(row, col, projTri);
        if (depth < zBuff[row * NUM_COL + col]) {
          shade(row, col, projTri);        
          zBuff[row * NUM_COL + col] = depth;
        }
      }
    }
  }
}
```

Let us dig a bit more into what should happen in that `shade` function in the next section.

### How Texturing Works

OK! Now we know which pixels needs shading, but we don't know what colour to actually chose and
how...

There are many factors at play here, but we are solely focusing on texturing, *i.e.* mapping a 2D
image/pattern onto our 3D model.

The formal way to look at this is through surface parametrisation. You can find an overview
[here](https://graphics.stanford.edu/courses/cs468-05-fall/Papers/param-survey.pdf)
if you are interested and willing. Knowing the theory is definitely useful, especially
for its other notable uses in graphics and modeling, such as parametrised curves and
surfaces like [Bézier curves](https://en.wikipedia.org/wiki/B%C3%A9zier_curve) and
[NURBS](https://en.wikipedia.org/wiki/Non-uniform_rational_B-spline)) which are ubiquitous in these
domains.

Here, however, here we are only interested in mapping textures onto triangles. So we'll take a
shortcut.

We need to add some additional data to our triangles:

```c
typedef struct vec2 {
  float u;
  float v;
} vec2;

typedef struct tri3D {
  vec3 pos[3];
  vec2 tex[3]; // extend struct with texture coordinates
} tri3D;
```

Now, each triangle vertex is mapped to a 2D coordinate, *i.e.* a specific point on an image. This
is enough to define a mapping to the whole triangle. It can be done in the following way:

Let the vertices \\(A\\), \\(B\\), and \\(C\\) (in world space) have texture coordinates \\(t_A\\),
\\(t_B\\), and \\(t_C\\) respectively. Let \\(P\\) be a point in the triangle with barycentric
coordinates \\(\left(m_A, m_B, m_C \right).\\) We can interpolate the texture coordinate at \\(P\\),
named \\(t_P\\) as follows:

\\[
t_P = m_A t_A + m_B t_B + m_C t_C.
\\]

There are reasons, beyond computational limitations, why doing this is fundamentally a good way to
interpolate the texture coordinates in our context. Discussing this would go beyond the scope of
this blog post, so [here is an entry point](https://en.wikipedia.org/wiki/Affine_transformation) for
whoever wants to dig deeper.

## The PlayStation Texture Coordinates Computation

As it turns out, the Playstation had to resort to a less-than-ideal approximation of the texture
interpolation because of the hardware limitations of the time. Let us detail what this approximation
looks like and why it yields "wobbly" results.

### Affine Texture Interpolation

Let us fix some notations:

* Let \\(ABC\\) be a non-degenerate triangle in 3D space (and in the positive half-space
\\( z > 0.\\))
* Let \\(\overline{A}\overline{B}\overline{C}\\) be its projection on screen space.
* Let \\(a\\), \\(b\\), and \\(c\\) be the respective texture coordinates of \\(A\\), \\(B\\),
and \\(C.\\)
* Let \\(\overline{P}\\) be a pixel coordinates in screen space.
* Let \\(\left(m_{\overline{A}}, m_{\overline{B}}, m_{\overline{C}}\right)\\) be the barycentric
coordinates of \\(\overline{P}\\) *w.r.t.* triangle \\(\overline{A}\overline{B}\overline{C}.\\)

The affine interpolation of texture coordinates \\(p\\) for the pixel at \\(\overline{P}\\) is given
by:

\\[
p =  m_{\overline{A}} a + m_{\overline{B}} b + m_{\overline{C}} c,
\\]

which doesn't necessarily, at first glance, seem crazy. But upon closer examination, the issue is
that we compute the interpolation of the texture coordinates using the barycentric coordinates in
__*2D screen space*__ and not __*3D world space*__.

This matters a great deal in our case, since we used a perspective projection to go from 3D to 2D
space and as such, the barycentric coordinates in 2D have no reason to be the same as the 3D space
one.

Additionally, the texture coordinates do not change linearly *w.r.t* vertex coordinates \\(x\\),
\\(y\\), and \\(z\\) but rather \\(x\\), \\(y\\), and \\(w = \frac{1}{z}\\) in that projected space.
This is because the perspective projection itself is not linear. This particular property makes the
approximation fundamentally flawed.

The following sketch should help illustrate the issue, from 2D to 1D space:

![Affine Projection in 2D](affine-projection-2d.png)

From this sketch, one can see that \\(\overline{P}\\) is the middle of \\(\left[\overline{A},
\overline{B}\right]\\), or, in other words, its barycentric coordinates are \\(\left(\frac12,
\frac12\right).\\) Its pre-image \\(P\\) under the perspective projection, however, is not the
middle of \\(\left[A, B\right]\\) and thus has different barycentric coordinates altogether.

So how do we fix this?

### The Perspective-Correct Interpolation

As mentioned, in the previous subsection, the "wobbliness" stems from the barycentric coordinates
in 3D and 2D space not coinciding. Let us show how they are actually linked, and what that means for
the texture coordinate computation with as elementary a computation as possible. Start with the 3D
triangle \\(ABC\\) which is both non-degenerate and lies in the positive half-space.

\\[
\left\\{(x, y, z), \\, (x,y) \in \mathbb{R}^2 \text{ and } z > 0\right\\}.
\\]

Let \\(P\\) be a point in the triangle \\(ABC\\) with barycentric coordinates \\(\left(m_A,
m_B, m_C \right).\\) Let \\(\overline{A}\overline{B}\overline{C}\\) be the projected triangle
on the screen and \\(\overline{P}\\) be the projection of \\(P\\) onto that same screen. Let
\\(\left(m_{\overline{A}}m_{\overline{B}}m_{\overline{C}}\right)\\) be the barycentric coordinates
of \\(\overline{P}\\) in \\(\overline{A}\overline{B}\overline{C}.\\)

Let us first write:

\\[
m_A A + m_B B + m_C C = P
\\]

which implies that

\\[
\left\\{\begin{aligned}
& z_A m_A \overline{A} + z_B m_B \overline{B} +z_C m_C \overline{C} = P \\\
& \text{and} \\\
& z_A m_A + z_B m_B + z_C m_C = z_P
\end{aligned}\right.
\\]

which, in turn, implies that

\\[
\left\\{\begin{aligned}
& \frac{z_A m_A}{z_P} \overline{A} + \frac{z_B m_B}{z_P} \overline{B} + \frac{z_C m_C}{z_P}
\overline{C} = \overline{P} \\\
& \text{and} \\\
& \frac{z_A m_A}{z_P} + \frac{z_B m_B}{z_P} + \frac{z_C m_C}{z_P} = 1
\end{aligned}\right.
\\]

hence
\\[
{m_{\overline{A}}} = \frac{z_A m_A}{z_P} ,\\,
{m_{\overline{B}}} = \frac{z_B m_B}{z_P} ,\\,
{m_{\overline{C}}} = \frac{z_C m_C}{z_P}.
\\]

From here on, it is clear that if we wish to interpolate the texture coordinates of point
\\(\overline{P}\\), we need to compute the depth \\(z_P\\) so as to infer the barycentric
coordinates of point \\(P\\) in 3D space. Its inverse is easily computed as

\\[
\frac{1}{z_P} = \frac{m_{\overline{A}}}{z_A} + \frac{m_{\overline{B}}}{z_B} +
\frac{m_{\overline{C}}}{z_C}.
\\]

This is a trivial consequence of

\\[
m_A + m_B + m_C = 1
\\]

and the previous identities between the barycentric coordinates in 2D screen space and 3D world
space.

In summary, the computation of the texture coordinates for a given pixel in a given triangle should
look somewhat like this:

```c

vec2 get_tex_coords(int pixRow, int pixCol, tri3D pTri) {
  // maps display to screen coordinates
  vec3 pScrCoords = screen_coords(pixRow, pixCol) 

  // Compute barycentric coordinates of pixel in screen space
  float sAreaInv = 1.f / s_area2D(pTri.pos[0], pTri.pos[1], pTri.pos[2]);

  float m0 = sAreaInv * s_area2D(pScrCoords, pTri.pos[1], pTri.pos[2]);
  float m1 = sAreaInv * s_area2D(pTri.pos[0], pScrCoords, pTri.pos[2]);
  float m2 = sAreaInv * s_area2D(pTri.pos[0], pTri.pos[1], pScrCoords);

  // Divide by the depth of the vertices (which we precomputed and stored on the
  // z component)
  m0 *= pTri.pos[0].z; 
  m1 *= pTri.pos[1].z; 
  m2 *= pTri.pos[2].z; 

  // Compute the depth of the pixel in 3D space
  float z = 1.f / (m0 + m1 + m2);

  // And now compute the components of each texture coordinates
  float u = z * (m0 * pTri.tex[0].u + m1 * pTri.tex[1].u + m2 * pTri.tex[2].u);
  float v = z * (m0 * pTri.tex[0].v + m1 * pTri.tex[1].v + m2 * pTri.tex[2].v);
  
  return (vec2) {u, v};
}
```

## Conclusion

It is really fascinating to me how hardware limitations, dictated by the technologies and costs
of the time, led designers and engineers to make such trade-offs. They, arguably, made the games
look worse at the time, but through the console achieving cult status, the screen space affine
interpolation of textures became a distinct look that we now find endearing.

So much so in fact that, should you want to create a game or an animation which emulates that look,
you now know one piece of the puzzle. You could, like me, write a software renderer which suits your
need, or, if you wanted to go the GPU accelerated way use some shader trickery to achieve the same
effect. You essentially need to revert the perspective projection by multiplying by the weights.

And should you want to push it even further: this was not the only notable sacrifice that the
Playstation engineers had to make in the name of cost and performance optimisation. As mentioned
earlier, the hardware lacked a z-buffer --- which seems wild to me considering this was a 3D first
console --- as well as... floating points!

So, on top of the *wobble*, the computations had to be made with fixed points which did not allow
for sub-pixel precision, which explains the *jitter* of Playstation games too. It also makes me
wonder whether the lack of precision would have made a perspective correct interpolation not doable
or not look better enough to justify the computational cost. 

Now, I'll go play Crash Bandicoot again, admire the jitter and wobble, and get sad once more at the
realisation that my reflexes really are not what they used to be.
