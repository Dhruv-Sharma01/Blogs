---
layout: post
title: "Understanding Dual Contouring"
date: 2024-02-18
categories: Isosurfacing
---

# **A Guide to Dual Contouring: From Theory to Implementation**

## **Introduction**

Marching Cubes is a well-known isosurface extraction algorithm that works well for generating smooth surfaces from volumetric data. However, it lacks the ability to accurately preserve sharp features. To overcome this limitation, I explored **Dual Contouring (DC)**, a technique that improves feature representation while maintaining efficiency.

After studying the concept, I implemented **Dual Contouring** in C++. While there are excellent tutorials by Paul Bourke on this topic, I found some parts of the standard approach to be redundant. To optimize the implementation, I replaced certain structures with **lookup hash maps**, which improved retrieval efficiency at the cost of increased memory usage.

This blog covers the step-by-step process of my **C++ implementation**, from defining an implicit grid to vertex placement using **Quadric Error Functions (QEF)**.

---

## **Defining the Implicit Grid**

The first step in Dual Contouring is defining a **scalar field** that represents our implicit surface. I created a **bounding box** with dimensions **8×8×8** in the x, y, and z directions. The implicit function defines the shape of the surface within this bounding box.

The octree-based Signed Distance Field (SDF) for this domain is then constructed, providing an efficient way to handle surface intersections and ensure adaptive subdivision where necessary.

---

## **Octree Representation**

To store the hierarchical structure of our **adaptive grid**, we use an **octree**. This allows efficient storage of volumetric data and helps reduce computational complexity.

### **Cell Structure in the Octree**

Each **cell** in the octree stores:

- **Coordinates** `(x, y, z)` representing the cell center.
- **halfsize**, the distance from the center to the cell boundary.
- **generator**, a pointer to the function defining the implicit surface.
- **children**, an array of 8 pointers to its subdivided cells.
- **isleaf**, a boolean flag to determine if the cell is a leaf.
- **vertex**, the computed **dual vertex** of the cell.

```cpp
class cell {
public:
    float x, y, z, halfsize;
    float (*generator)(float, float, float);  // Function pointer to evaluate the implicit function.
    bool isleaf;
    cell* children[8];
    vec3 vertex;

    // Default constructor
    cell() : x(0), y(0), z(0), halfsize(0), generator(nullptr), isleaf(false), vertex(INT_MIN, INT_MIN, INT_MIN) {
        for (int i = 0; i < 8; i++) children[i] = nullptr;
    }

    // Constructor with generator function
    cell(float x, float y, float z, float halfsize, float (*generator)(float, float, float))
        : x(x), y(y), z(z), halfsize(halfsize), generator(generator), isleaf(true), vertex(0, 0, 0) {
        for (int i = 0; i < 8; i++) children[i] = nullptr;
    }
};
```

### **Subdivision Strategy**

The **homogeneous()** function determines if a cell is **homogeneous** (i.e., all its corners share the same sign). If not, the cell is subdivided further using **recursive subdivision** until:

1. A predefined **minimum cell size** is reached.
2. The cell becomes **homogeneous**.

```cpp
void recursiveSplit(float minHalfSize) {
    if (halfsize <= minHalfSize || homogeneous()) return;
    split();
    for (int i = 0; i < 8; i++) if (children[i]) children[i]->recursiveSplit(minHalfSize);
}
```

---

## **Placing the Dual Vertex Using QEF Optimization**

Unlike Marching Cubes, **Dual Contouring places a vertex per cell**. This vertex is computed by minimizing the **Quadric Error Function (QEF)**, which helps accurately determine the best-fit position for **sharp features**.

### **Steps to Compute the Dual Vertex**

1. Identify **edge intersections** within the octree cell.
2. Compute **normals** at these intersections.
3. Solve for the optimal vertex **v** using **QEF minimization**.

If `QR = false`, the **determinant method** is used to solve the QEF. Otherwise, **Singular Value Decomposition (SVD)** is applied for a more robust solution.

### **QEF Formulation and SVD Solution**

If `QR = true`, the function uses **Singular Value Decomposition (SVD)** to solve the least-squares problem for minimizing the **Quadric Error Function (QEF)**.  

### Matrix Formulation  

1. Construct an **A matrix** where each row represents the **normal vector** at an intersection.  
2. Construct a **d vector**, where each entry is the **dot product** of the normal and the intersection point.  

\[A \cdot v = d\]

Where:  

- \( A \) is an \( m \times 3 \) matrix of normal vectors.  
- \( d \) is an \( m \times 1 \) vector, where each entry is \( n_i \cdot p_i \).  

### SVD Optimization  

- Solve for \( v \) in the least-squares sense:  

\[v = A^+ d\]

- Here, \( A^+ \) is the **pseudoinverse**, computed using **SVD decomposition**:  

\[A = U S V^T\]

- Using **Eigen’s JacobiSVD**, the solution is computed efficiently.  

### Final Vertex Placement  

- The computed **\( v \)** is stored as the **dual vertex** for the octree cell, representing the best approximation of the **feature point**.  



---

## **Polygonization Strategy**

After placing the dual vertices, we need to construct a mesh by connecting them.
For the polygonisation the original paper connects the dual vertices of cells surrounding minimal edges, i.e. edges that represent a sign change and do not properly contain edge of neighbouring leaf cell. In our simplified implementation we are not doing bottom up collapsing of those leaf whose combined QEF residual will be tolerable. That's why we used intersecting edges of leaf cells and stored their neighbouring dual vertices in unordered map to create triangulations.We connect the dual vertices of those cells that share a common intesecting edge to form triangles.

---

## **Conclusion**

Dual Contouring significantly improves upon Marching Cubes by preserving sharp features while maintaining efficiency. Using **adaptive octrees**, **QEF optimization**, and **efficient data structures**, we can construct high-quality isosurfaces efficiently.

The complete **C++ implementation** is available [here](https://github.com/Dhruv-Sharma01/Adaptive-Dual-Contouring-over-Uniform-Leaf). Future optimizations may include **adaptive merging of dual vertices** and **GPU acceleration for real-time processing**.
