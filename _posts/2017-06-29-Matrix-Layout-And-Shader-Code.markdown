---
layout: 	post
title:  	"Matrix Layout And Shader Code"
date:   	2017-06-29 22:18:46 -0500
category: 	"Graphics"
published:	false
---

For any graphics programmer, matrix math usually becomes second nature, however, I recently came across some sub-optimal shader code that made me stop and think about this. Things can get confusing real quick when you have to consider differences such as row-major vs. column-major, row vectors vs. column vectors, etc. Code bases are sometimes inconsistent, languages differ in their storage conventions, and sometimes there are incosistencies even within the same API (for example, HLSL uses column major storage but the float4x4 constructor takes row vectors as input). Morever, this is a scenario where 2 wrongs *can* make a right but you can run into some performance pitfalls in the process. 

$$
\begin{pmatrix} 
m_{11} & m_{12} \\
m_{21} & m_{22} 
\end{pmatrix}
$$

$$ 
\left[
    \begin{array}{cc|c}
      1&2&3\\
      \hline
      4&5&6
    \end{array}
\right] 
$$