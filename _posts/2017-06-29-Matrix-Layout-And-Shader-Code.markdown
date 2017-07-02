---
layout: 	post
title:  	"Matrix Layout And Optimization"
date:   	2017-06-29 22:18:46 -0500
category: 	"Graphics"
published:	false
---

For any graphics programmer, matrix math usually becomes second nature, however, I recently came across some sub-optimal shader code that made me stop and think about this. Things can get confusing real quick when you have to consider differences such as row-major vs. column-major, row vectors vs. column vectors, etc. Code bases are sometimes inconsistent, languages differ in their storage conventions, and sometimes there are incosistencies even within the same API (for example, HLSL uses column major storage but the float4x4 constructor takes row vectors as input). The choices you make can effect both correctness and performance.

It is impornant to understand both matrix storage and vector matrix products. Fabian Giesen has a [nice writeup](https://fgiesen.wordpress.com/2012/02/12/row-major-vs-column-major-row-vectors-vs-column-vectors/) on this, but here is a gist.

### Matrix Storage

Row-major and colum-major order refer the way that elements of the matrix (a 2-dimensional quantity) are stored contiguously in memory (which is 1-dimensional). Consider the following matrix :

$$
\begin{bmatrix} 
m_{11} & m_{12} & m_{13}\\
m_{21} & m_{22} & m_{23} \\
m_{31} & m_{32} & m_{33} \\
\end{bmatrix}
$$

**Row-Major** stores the elements as $$m_{11},m_{12},m_{13},m_{21},m_{22},m_{23},m_{31},m_{32},m_{33}$$. C++ uses Row-Major storage. So, if you have an array representation for your matrix such as `float M[3][3]` in your application, this is how the elements are stored in memory.  

**Column-Major** stores the elements as $$m_{11},m_{21},m_{31},m_{12},m_{22},m_{32},m_{13},m_{23},m_{33}$$. HLSL uses column major storage by default [^fn1]. 

So, if you have a matrix in C++ and pass the memory blob of the matrix to HLSL, the rows are read in as columns. The matrix is essentially [***transposed***](https://en.wikipedia.org/wiki/Transpose). 

### Vector Matrix Product

When you transform a vector by a matrix, it is basically a matrix multiplication of a $$n$$ x $$n$$ matrix with vector which is represented either as a $$n$$ x $$1$$ matrix such as $$ \begin{bmatrix} v_1 & v_2 & v_3 & \cdots \end{bmatrix} $$ or as a $$1$$ x $$n$$ matrix such as $$ \begin{bmatrix} v_1 \\ v_2 \\ v_3 \\ \vdots \end{bmatrix}$$. The first vector representation is referred to as **Row Vector** and and latter one is a **Column Vector**.

For the multiplication to be valid, *row vectors* can only be pre-multiplied. For a matrix $$M$$ and vector $$\vec{v}$$, a row vector multiplication is defined as $$\vec{v}M$$.

Conversely, *column vectors* can only be post-multiplied, such as $$M\vec{v}$$.

Now, let's look at the row vector matrix multiplication for a second.

$$\vec{v}M = 
\vec{v}
\left[
	\begin{array}{c|c|c}
		m_{11} & m_{12} & m_{13}\\
		m_{21} & m_{22} & m_{23} \\
		m_{31} & m_{32} & m_{33} \\
	\end{array}
\right]  =
\begin{bmatrix}
\vec{v}\bullet\vec{M_1} & \vec{v}\bullet\vec{M_2} & \vec{v}\bullet\vec{M_3}
\end{bmatrix}
$$

*The above basically says that matrix vector multiplication can be represented as a dot product ($$\bullet$$) of the vector with vectors composed of each column of the matrix $$M$$ ($$\vec{M_1}$$, $$\vec{M_2}$$, $$\vec{M_3}$$)*. (To avoid confusion, I am not going to refer to $$\vec{M_1}$$, $$\vec{M_2}$$ and $$\vec{M_3}$$ as column vectors. It is also incorrect to call $$M$$ a column matrix or such.)

### HLSL Vector Transpose Equivalence

If, $$\vec{s} = \vec{v}M$$, then


$$
\vec{s}^T = {(\vec{v}M)}^T = M^T \vec{v}^T
$$

Now, in a strictly mathematical sense, 

$$\vec{v}^T \neq \vec{v}$$


However, HLSL does not distinguish between row and column vectors. A `float4` can represent either one, and it is the context in which it is used (PreMultiply or PostMultiply), that determines whether it is treated as a row or column vector.

So, for HLSL math we can write

$$
\vec{v}M = M^T \vec{v}
$$

This is important to note. Combined with the fact that the HLSL matrices are transposed (implicitly) when passing in from C++, this means there are two ways we can perform our matrix math in the shader. From here on out, I am going to assume that the multiplication order on the host/C++ side is $$\vec{v}M$$.

* **Case 1 :** Pass the matrix as it is on the C++ side and change order of multiplication in the shader to $$M^T \vec{v}$$
* **Case 2 :** Perform an additional (explicit) transpose of the matrix on the C++ side before passing it to HLSL and retain the same vector matrix multiplication order $$\vec{v}M$$. This is because $$(M^T)^T = M$$

However, are the two approaches actually equivalent? Let's look at the register contents in the shader and the associated asm code.

#### Case 1 

Each column of the HLSL matrix is loaded into separate register

[![img1](/images/CB-Matrix.jpg)](/images/CB-Matrix.jpg)


[^fn1]: [Microsoft - Matrix Ordering](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509634(v=vs.85).aspx#Matrix_Ordering)