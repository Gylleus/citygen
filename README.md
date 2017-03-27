# Introduction

This page will be used as documentation of method process and results for my thesis of generating virtual urban environments. I will implement a split grammar based procedural generator and attempt an approach at an inverse procedural one. I will be using Unity3D as an environment for the project.

### Split Grammar

The split grammar that I will use splits and replaces shapes into other shapes, often smaller pieces. How and when such a split is performed is defined by rules. A simple example of three rules would be:



```markdown
- Start -> split(Y) {0.7: Ground | N: Floors | 1: Top} [minY: 3]
- Floors -> split(Y) {0.75: Floor | N: Floor} [minY: 1.3]
- Floors -> split(Y) {N: Floor} [maxY: 1.3]
```

Here the start component would be the initial shape that the algorithm receives. If it has a height of at least 3 (Y >= 3) the first rule is valid and can be applied, splitting the initial shape into a ground, floor and top component. The floor component has additional rules which allows it to be split recursively until its Y value is less than 1.3, at which point no more shapes can be split. This can also be described as all regions having reached a terminal state. 

The rule takes as input on shape as input to the left and inside the brackets the shapes that it will be split into are defined. The vertical line seperates different shapes. The numerical value in front of the shape name defines the size of it. It can also be defined as 'N' meaning that the shape is flexible and can be changed in size to better fit the size of the input shape.

This simple example of rules were inspired by https://cs.uwaterloo.ca/research/tr/2009/CS-2009-23A.pdf.

We can avoid using such recursive rules by defining a repeat rule that will continue creating shapes to fill out the input shapes space. The floor rules described above can thus be defined as:


```markdown
- Floors -> repeat(Y) {0.75N: Floor}
```
I have also defined rules that can work in different dimension, where it chooses the dimensions largest in size of the input shape.

```markdown
- Floor -> repeat(XZ) {0.25N: Facade | 0.25: WindowArea | 0.25N: Facade}
```

When the above rule is to be executed the grammar derivator will look at the current shape to see whether its X or Z size is larger. The greater one will be chosen as the splitting axis.

### Initial results
![Grey Building](/citygen/images/grey-building.png)
