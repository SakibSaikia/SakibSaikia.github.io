---
layout: 	post
title:  	"Accurate Sun Position for Time of Day"
date:   	2023-01-12 00:00:00 -0500
category: 	"Graphics"
published:	true
---

Time of Day systems simulate the movement of the sun across the sky. There are several challenges to implement such a system, but this post deals with the most fundamental of these - how to generate the sun position in the sky. While an accurate position is not required, it is fairly simple to generate and very satisfying. I came across this in ["A Practical Analytic Model for Dylight"](https://courses.cs.duke.edu/cps124/fall02/resources/p91-preetham.pdf) by Preetham et. al. and wanted to extract the relevant portion here for ease of discoverability.

The three parameters that determine the position of the sun in the sky are:

* Solar time $$t$$ in decimal hours
* Julian date $$J$$ which is the day of the year in [1,365] range
* Latitute $$l$$ of the site in radians $$[-\frac{\pi}{2}, \frac{\pi}{2}]$$

Solar time is the calculation of time based on the position of the sun, and it deviates from standard time by a few seconds based on the Julian date and site location. If you want to work with standard time, the paper provides a conversion but the difference is not significant unless you want an exact reproduction.

The first step is to calculate the [solar declination](https://en.wikipedia.org/wiki/Position_of_the_Sun#Declination_of_the_Sun) angle ($$\delta$$). This is the angle between the Earth-Sun line and the equatorial plane. It reaches a max value of $$23.44^\circ$$ on the June solstice and a minimum angle of $$-23.44^\circ$$ on the December solstice. It is approximated by the following formula:

$$\delta = 0.4093 sin(\frac{2\pi(J - 81)}{368})$$

The sun elevation ($$\theta$$) and azimuth ($$\phi$$) can then be computed as follows:

$$\theta_s = \frac{\pi}{2} - \arcsin(\sin l\sin\delta - \cos l\cos\delta\cos\frac{\pi t}{12})$$

$$\phi = \arctan(\frac{-\cos\delta\sin\frac{\pi t}{12}}{\cos l\sin\delta - \sin l\cos\delta\cos\frac{\pi t}{12}})$$

Code snippet for the above:

```
Vector3 GetSunDirection(float t, int J, float l)
{
	// Solar declination
	float delta = 0.4093f * sin(PI * (J - 81.f) / 368.f);

	float sin_l = sin(l);
	float cos_l = cos(l);
	float sin_delta = sin(delta);
	float cos_delta = cos(delta);
	float t = PI * t / 12.f;
	float sin_t = sin(t);
	float cos_t = cos(t);

	// Elevation
	float theta = 0.5f * PI - asin(sin_l * sin_delta - cos_l * cos_delta * cos_t);

	// Azimuth
	float phi = atan(-cos_delta * sin_t / (cos_l * sin_delta - sin_l * cos_delta * cos_t));

	// Sun dir based on Time of Day
	Vector3 sunDir;
	sunDir.x = sin(theta) * cos(phi);
	sunDir.z = sin(theta) * sin(phi);
	sunDir.y = cos(theta);
	m_sunDir.Normalize();

	return sunDir;
}
```

<p align="center">
<div class="embed-container">
  <iframe
      src="/videos/time-of-day.mp4"
      width="640"
      height="360"
      frameborder="0"
      scrolling="no"
      allowfullscreen="true">
  </iframe>
</div>
</p>