---
layout: 	post
title:  	"Deriving Lambertian BRDF from first principles"
date:   	2019-08-19 23:09:46 -0500
category: 	"Graphics"
published:	false
---

This is a short exercise on integration. We will use it to derive the Lambertian BRDF from first principles.

It is widely accepted that the Lambertian BRDF is $$\frac{Albedo}{\pi}$$. Let's see how we get there from the rendering equation below:

$$
L_o(x,\omega_o) = L_e(x,w_o) + \int_{\Omega}f_r(x,\omega_i,\omega_o)  L_i(x,\omega_i)  (\omega_i . n) d\omega_i
$$

where,

$$L_o$$ = Outgoing radiance at point $$x$$ along the direction given by solid angle $$\omega_o$$  
$$L_e$$ = Emitted radiance at point $$x$$ along the direction given by solid angle $$\omega_o$$  
$$L_i$$ = Incoming radiance at point $$x$$ along the direction given by solid angle $$\omega_i$$  
$$f_r$$ = BRDF at point $$x$$ for incoming radiance along $$\omega_i$$ and outgoing radiance along $$\omega_o$$  
$$(\omega_i . n)$$ = Weakening factor  
$${\Omega}$$ = This is used to denote the integration over the hemisphere

For the purpose of this discussion, we are interested in diffuse reflectance only. So, we can drop the emissive term. Also, since the Lambertian BRDF is a model of diffuse reflectance it is invariant to the viewing direction. In other words, it is constant and can be taken out of the itegral. Let's call this constant albedo $$\alpha$$ for now. Later we will derive the actual value for the BRDF to make it energy-preserving. 

So, our simplified rendering equation looks like this now.

$$
L_o(x) = \alpha L_i(x) \int_{\Omega}(\omega_i . n) d\omega_i \\
\Rightarrow \frac{L_o(x)}{L_i(x)} = \alpha \int_{\Omega}(\omega_i . n) d\omega_i
$$

By definition, a BRDF is the ratio of outgoing radiance to incoming radiance. Hence,

$$BRDF = \frac{L_o(x)}{L_i(x)} = \alpha \int_{\Omega}(\omega_i . n) d\omega_i$$

Still with me? None of this is new, and chances are you've seen this a hundred times -- the purpose of this is to lay the groundwork for our little integration exercise. Now, let's look at how to compute the integral $$\int_{\Omega}(\omega_i . n) d\omega_i$$ which is the Lambertian BRDF we set out to evaluate.










