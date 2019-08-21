---
layout: 	post
title:  	"Deriving Lambertian BRDF from first principles"
date:   	2019-08-19 23:09:46 -0500
category: 	"Graphics"
published:	false
---

This is a short exercise on integration. We will use it to derive the Lambertian BRDF from first principles.

It is already known that the Lambertian BRDF is $$\frac{Albedo}{\pi}$$. Let's see how we get there from the rendering equation below:

$$
\begin{equation}
L_o(x,\omega_o) = L_e(x,w_o) + \int_{\Omega}f_r(x,\omega_i,\omega_o)  L_i(x,\omega_i)  (\omega_i . n) d\omega_i
\tag{1}
\end{equation}
$$

where,

$$L_o$$ = Outgoing radiance at point $$x$$ along the direction given by solid angle $$\omega_o$$  
$$L_e$$ = Emitted radiance at point $$x$$ along the direction given by solid angle $$\omega_o$$  
$$L_i$$ = Incoming radiance at point $$x$$ along the direction given by solid angle $$\omega_i$$  
$$f_r$$ = Reflectance at point $$x$$ for incoming radiance along $$\omega_i$$ and outgoing radiance along $$\omega_o$$  
$$(\omega_i . n)$$ = Weakening factor for incoming direction $$\omega_i$$ and normal $$n$$  
$${\Omega}$$ = This is used to denote the integration over the unit hemisphere

For the purpose of this discussion, we are interested in diffuse reflectance only. So, we can drop the emissive term. Also, since the Lambertian BRDF is a model of diffuse reflectance it is invariant to the viewing direction. In other words, it is constant and can be taken out of the itegral. Let's call this constant albedo $$\alpha$$ for now. Later we will derive the actual value for the BRDF to make it energy-preserving. 

So, our simplified rendering equation looks like this now.

$$
L_o(x) = \alpha L_i(x) \int_{\Omega}(\omega_i . n) d\omega_i \\
\Rightarrow \frac{L_o(x)}{L_i(x)} = \alpha \int_{\Omega}(\omega_i . n) d\omega_i
$$

By definition, a BRDF is the ratio of outgoing radiance to incoming radiance. Hence,

$$
\begin{equation}
BRDF = \frac{L_o(x)}{L_i(x)} = \alpha \int_{\Omega}(\omega_i . n) d\omega_i
\tag{2}
\end{equation}
$$

Still reading? None of this is new, and chances are you've seen this a hundred times -- the purpose of this is to lay the groundwork for our little integration exercise. If, on the other hand, you are new to the topic I would refer you to the explanations in the Real Time Rendering books. There are are several excellent explanations [online](https://www.gamedev.net/blogs/entry/2260986-the-rendering-equation/). 

Now, let's look at how to compute the integral $$\int_{\Omega}(\omega_i . n) d\omega_i$$ which is the Lambertian BRDF we set out to evaluate. But, first let's do a quick refresher on integral calculus.

Informally, integration is the process by which we divide a region into *tiny parts* and then sum all those parts for the entire region to evaluate the property we are interested in such as area or volume.

<p align="center">
	<img src="/images/lambertian-brdf/area-rectangle.png">
</p>

So, let's start with computing the area of a rectangle using integration. Here, we define a rectangle in Catesian coordinates. We identify a small patch with length $$dx$$ and height $$dy$$. The area of this patch is $$dx.dy$$. Since there are 2 variables, we need to compute a double integral for the range of $$x$$ and $$y$$ respectively.

$$
RectangleArea = \int_{x=2}^6\int_{y=1}^4dxdy = \int_{x=2}^6dx\int_{y=1}^4dy = \Big[x\Big]_{x=2}^6 \Big[y\Big]_{y=1}^4 = (6-2)(4-1) = 12
$$

This checks out since the area of the rectangle for length 4 and height 3 is 12. Note that this area doesn't change under [rigid transforms](https://en.wikipedia.org/wiki/Rigid_transformation). We could bend this to a curved shape and the area would be same. Conversely, we could straighten a curved patch into a rectangle and compute the area that way. We will come back to this later.

Next up, let's try to compute the area of a circle.














