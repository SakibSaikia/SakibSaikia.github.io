---
layout: 	post
title:  	"Matrix Layout And Optimization"
date:   	2017-06-29 22:18:46 -0500
category: 	"Graphics"
published:	false
---

For any graphics programmer, matrix math usually becomes second nature, however, I recently came across some sub-optimal shader code that made me stop and think about this. Things can get confusing real quick when you have to consider differences such as row-major vs. column-major, row vectors vs. column vectors, etc. Code bases are sometimes inconsistent, languages differ in their storage conventions, and sometimes there are incosistencies even within the same API (for example, HLSL uses column major storage but the float4x4 constructor takes row vectors as input). Morever, this is a scenario where 2 wrongs *can* make a right but you can run into some performance pitfalls in the process. 

Fabian Giesen has a [nice writeup](https://fgiesen.wordpress.com/2012/02/12/row-major-vs-column-major-row-vectors-vs-column-vectors/) on this, but here is a gist.

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

So, if you have a matrix in C++ and pass the memory blob of the matrix to HLSL, the rows are read in as columns. The matrix is essentially [***transposed***](https://en.wikipedia.org/wiki/Transpose). This is important and we will come back to it, but first let's discuss matrix/vector multiplication.

When you transform a vector by a matrix, it is basically a matrix multiplication of a $$n$$ x $$n$$ matrix with vector which is represented either as a $$n$$ x $$1$$ matrix such as $$ \begin{bmatrix} v_1 & v_2 & v_3 & ... \end{bmatrix} $$ or as a $$1$$ x $$n$$ matrix such as $$ \begin{bmatrix} v_1 \\ v_2 \\ v_3 \\ ... \end{bmatrix}$$. The first representation is referred to as **Row Vector** and and latter one is a **Column Vector**.

For the multiplication to be valid, *row vectors* can only be pre-multiplied. For a matrix $$M$$ and vector $$\vec{v}$$, a row vector multiplication is defined as $$\vec{v}.M$$.

Conversely, *column vectors* can only be post-multiplied, such as $$M.\vec{v}$$.

Now, let's look at the row vector matrix multiplication for a second.

$$\vec{v}.M = 
\begin{bmatrix} v_1 & v_2 & v_3\end{bmatrix} * 
\begin{bmatrix} 
m_{11} & m_{12} & m_{13}\\
m_{21} & m_{22} & m_{23} \\
m_{31} & m_{32} & m_{33} \\
\end{bmatrix} =
\begin{bmatrix} v_1 & v_2 & v_3\end{bmatrix} * 
\left[
	\begin{array}{c|c|c}
		m_{11} & m_{12} & m_{13}\\
		m_{21} & m_{22} & m_{23} \\
		m_{31} & m_{32} & m_{33} \\
	\end{array}
\right]  =
\begin{bmatrix}
\vec{v}.\vec{M_1} & \vec{v}.\vec{M_2} & \vec{v}.\vec{M_3}
\end{bmatrix}
$$

***The above basically says that matrix vector multiplication can be represented as a dot product of the vector with vectors composed of each column of the matrix $$M$$ ($$\vec{M_1}$$, $$\vec{M_2}$$, $$\vec{M_3}$$)***. (To avoid confusion, I am not going to refer to $$\vec{M_1}$$, $$\vec{M_2}$$ and $$\vec{M_3}$$ as column vectors. It is also incorrect to call $$M$$ a column matrix or such.)

[^fn1]: [Microsoft - Matrix Ordering](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509634(v=vs.85).aspx#Matrix_Ordering)