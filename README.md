# Introduction

This page will be used as documentation of method process and results for my thesis of generating virtual urban environments. I will implement a split grammar based procedural generator and attempt an approach at an inverse procedural one. I will be using Unity3D as an environment for the project.

### Split Grammar

The split grammar that I will use splits and replaces shapes into other shapes, often smaller pieces. How and when such a split is performed is defined by rules. A simple example of three rules would be:



```markdown
- Start -> split(Y) {0.7: Ground | N: Floor | 1: Top} [minY: 3]
- Floor -> split(Y) {0.75: Terminal Floor | N: Floor} [minY: 1.3]
- Floor -> split(Y) {N: Terminal Floor} [maxY: 1.3]
```

Here the start component would be the initial shape that the algorithm receives. If it has a height of at least 3 (Y >= 3) the first rule is valid and can be applied, splitting the initial shape into a ground, floor and top component. The floor component has additional rules which allows it to be split recursively until its Y value is less than 1.3, at which point no more shapes can be split. This can also be described as all regions having reached a terminal state. This simple example of rules were inspired by https://cs.uwaterloo.ca/research/tr/2009/CS-2009-23A.pdf.

### Initial results

(image of building)
