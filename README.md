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

## Rules implemented

In addition to the split and repeat rule I have added a few other as well for both convenience and functionality. The first one is *decompose*. It will take a 3D shape and extract the faces of it to go down in dimensionality. Each face will originate from the initial shapes facing and then be rotated appropriately. This makes further work on them easier as their local XYZ-axes can be used instead of complicated global rules. An example is:

```markdown
- Start -> decompose(XZ) { Facade | Facade }
```

This will create a hollow box of four facades without roof or floor.

The next rule is *protrude* which scales and moves a shape. This can be used to protrude pillars or balconies forward, or a window backward, from the a facade giving it depth. This creates more realistic buildings rather than merely being a textured flat surface.

Example:

```markdown
- Pillar -> protrude(Z) { 0.5 }
```

I also added a rule called *replace* which will change the form of a shape. Initially shapes are assumed to be cubes but can be changed using this rule. At the moment there is only support for creating hexagons but I would like to add cylinders and prisms as well. The problem with cylinders however is that performing split rules on them is quite complicated due to the nature of its surface. It would have to be unwrapped and then wrapped again to simplify working on it, but Unity does not provide good support for this. The most likely option would be to use procedural mesh generation upon splitting and repeating, but that might be too time consuming for this project.


![Hexagon Example](/citygen/images/hexagonShape.png)

# Inverse generation

I have just started with my inverse generator as described in *Inverse Procedural Modeling of Facade Layouts*. My system will take two pictures as input. The first is the image of the facade and the second describes where the terminal regions of it are. The latter is defined by using rectangles of different colors. A rectangle will be a terminal region which are and position corresponds to the facade image. All rectangles of the same color are the same type of regions, such as all windows being blue rectangles for example. I can then use these rectangles to produce a grammar through the proposed algorithm.

![Facade](/citygen/images/20170324_180808.png)
![Terminal Regions](/citygen/images/TerminalRegions.png)

The example above is very simple where each floor and the area between them are defined as terminal regions.

All rectangles of the same color need to be of the same size. This can be quite tedious to create manually and some layouts may make this very hard. As such I likely need to be able to adjust and rescale areas of the facade if rectangles of the same color have different sizes. How they do this is not described well in the paper.

The goal is to find a description of splits, repeats, replacements and protrusions that matches the input facade. Splitting is the most trivial as we can merely look at how we can split the start area into smaller areas. This would not create a flexible layout however as there would be no repeat rules which would result in the facade not fitting well on shapes of other dimensions and sizes than the input facade. Thus we need some way to find which areas can be described by repeat rules.

To identify repeat-areas we need to find areas that only consists of sub-areas that occur at least twice. If an area contains an element or region that only appears once it should not be defined by a repeat but rather a split. To find these areas I follow an approach similar to the one described in Inverse Procedural Modeling of Facade Layouts, where they start by finding all terminal regions that occur twice. I then do a total search of all possible merges of these terminal regions to find all possible areas containing only repeating elements.

We then have a list of all areas that are composed of only repeating areas. During the splitting process we can then check which subareas (if any) are defined to be repeats. If there is one we can adjust the generated rules accordingly.

To define the protrusion of areas we can either give the system a height map of different areas, such that for example red areas should have a Z value of 2. Areas containing the red element will then be protruded accordingly. Another variant I am considering is just to ignore the protrusion and replacement parts during rule generation and instead handle them dynamically during the building generation process, but this might lead to an unintuitive user experience.

# Splitting

When splitting we have to decide if we want to do it on the X or Y axis. To do this I chose to split on the axis that involves the most amount of splits. If splitting an area on the X axis would result in 6 subareas and the Y axis 3 subareas then the X axis will be chosen. In the ideal case one axis will have 0 splits available and we can be certain that the other axis is the right one. If the region can not be split neither on X or Y and it does not simply contain one terminal region then the facade layout will be regarded as faulty and the rule generation will terminate.

To split a region into smaller regions I chose to try to draw a line across the region and see if it can be aligned with only the ends of terminal areas inside it. If it can it means we can do a clean separation from the original area and the one under the line. This process repeats until we have split the entire region into smaller parts. If no further split can be performed and there are still terminal regions not in any subregion then we will conclude that the region can not be split on this axis and return null.

When we have decided our axis to split and have a list of subregions it would contain we are ready to write the split rule. First we would like to check for repeats though which we can do by searching through our list of repeated areas and see which ones match the ones in the split. If these repeated areas are next to each other they can be merged and thus described by a repeat rule. 

We now have the information to write the split rule. If any repeated areas were found within the split we will name a temporary area inside the split which will contain a repeat rule. The resulting regions are then returned and added to the queue of regions to split. The rule generation will terminate when there are no more non-terminal regions to split and we will hopefully have obtained an appropriate grammar for the facade to use in our procedural generator.




