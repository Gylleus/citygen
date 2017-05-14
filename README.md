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

It is worth noting that all grammar needs to start with a decompose rule into the starting shape for the facade to go down in dimensionality from 3D to 2D.

# Splitting

When splitting we have to decide if we want to do it on the X or Y axis. To do this I chose to split on the axis that involves the most amount of splits. If splitting an area on the X axis would result in 6 subareas and the Y axis 3 subareas then the X axis will be chosen. In the ideal case one axis will have 0 splits available and we can be certain that the other axis is the right one. If the region can not be split neither on X or Y and it does not simply contain one terminal region then the facade layout will be regarded as faulty and the rule generation will terminate.

To split a region into smaller regions I chose to try to draw a line across the region and see if it can be aligned with only the ends of terminal areas inside it. If it can it means we can do a clean separation from the original area and the one under the line. This process repeats until we have split the entire region into smaller parts. If no further split can be performed and there are still terminal regions not in any subregion then we will conclude that the region can not be split on this axis and return null.

When we have decided our axis to split and have a list of subregions it would contain we are ready to write the split rule. First we would like to check for repeats though which we can do by searching through our list of repeated areas and see which ones match the ones in the split. If these repeated areas are next to each other they can be merged and thus described by a repeat rule. 

We now have the information to write the split rule. If any repeated areas were found within the split we will name a temporary area inside the split which will contain a repeat rule. The resulting regions are then returned and added to the queue of regions to split. The rule generation will terminate when there are no more non-terminal regions to split and we will hopefully have obtained an appropriate grammar for the facade to use in our procedural generator.

# Symmetry detection

To identify symmetry and define repeat rules I perform a search during each split. Since we have already decided to split on an axis we have a list of regions that are perfectly aligned on either X or Y. The goal is to find an area that consists merely of two or more identical subregions within the list of splits.

The algorithm for finding these areas starts with the list of splits. We then check for duplicate regions in the split as a region would have to appear at least twice for it to be considered for repetition. The algorithm appears in pseudocode below.

```markdown
- RR = empty
- DR = all split elements that have a duplicate counterpart
- While DR is not empty
-   try to merge each duplicate region with a repeated split element
-   remove all regions that no longer have duplicates from DR
-   attempt to merge together these duplicate areas
-   add such merged areas to RR
-   remove all regions that no longer have duplicates from DR

- Make sure that regions in RR do not overlap with each other
- return RR
```
The repeated areas that we find will be the merges of identical regions as that was our definition of a repeating area. The regions found are then return and used when writing the split rule and potential repeat rules. The final step to ensure that regions are not overlapping is important to make sure that each split region only belongs to one repeat area. For example, a sequence of ABABABA can be defined as a repeat area in two ways. (A)(BABABA) or (ABABAB)(A). If we do not remove overlapping regions these two will create conflicting logic when attempting check which repeat area a specific split region belongs to. Thus we have to choose one of them, which I do arbitrarily by removing all regions that overlaps the first sequence I find for each split region.

# The generator working

After a lot of tweaking the above algorithm finally works. As input a facade image and facade layout (image of terminal regions) need to be provided as stated above. The result will be a grammar that works with the previously implemented procedural generator.

As an example the facade layout provided above provides the following procedural description:

![Terminal Regions](/citygen/images/TerminalRegions.png)

```markdown
- Start -> decompose(XZ) {TestBuilding | TestBuilding}
- TestBuilding -> split(Y) {1.481481: TestBuilding1 | 3.981481N: TestBuilding2 | 0.3518519: TestBuilding3 | 1.148148: TestBuilding4}
- TestBuilding2 -> repeat(Y) {0.2962963: TestBuilding5 | 1.018518: TestBuilding6}
```
Quite simple and also makes sense. The first split rule creates 4 new regions: the green, red, yellow and the middle area containing purple and blue elements. The last mentioned is then defined by a repeat rule of blue and purple. This makes the building stretchable on the Y axis as dragging it out will allow for more repeated areas as the purple and blue area is defined as relative.

The names given by the algortihm is quite difficult for a human to read as they provide no information about the region, such as if it is a window or a wall nor does it tell outright which one is a terminal region. The first mentioned would be very hard to adjust as the algorithm would need to create contextual information from the material or structure of the facade. Alternatively user input could be provided for each terminal region to give them appropriate names. Showing which areas are terminal could be easily adjusted in the algorithm though if it would be of desire. But the grammar is mainly intended for the eyes of the computer and thus do not really need informative names unless for debugging and curiosity.

Another example of a more advanced facade layout and its result:


![Terminal Regions](/citygen/images/TerminalRegionsAdvanced2.png)

```markdown
- Start -> decompose(XZ) {TestBuilding | TestBuilding}
- TestBuilding -> split(Y) {6.546296: TestBuilding1 | 0.4351852: TestBuilding2}
- TestBuilding1 -> split(X) {3.105159: TestBuilding3 | 0.4166667: TestBuilding4 | 2.847222: TestBuilding5 | 0.4166667: TestBuilding4 | 3.164683: TestBuilding3}
- TestBuilding3 -> split(Y) {1.722222: TestBuilding6 | 4.083333N: TestBuilding7 | 0.7222222: TestBuilding8}
- TestBuilding7 -> repeat(Y) {0.9444445: TestBuilding8 | 0.4351852: TestBuilding9}
- TestBuilding5 -> split(Y) {1.722222: TestBuilding6 | 4.083333N: TestBuilding10 | 0.7222222: TestBuilding11}
- TestBuilding10 -> repeat(Y) {0.9444445: TestBuilding11 | 0.4351852: TestBuilding9}
- TestBuilding8 -> split(X) {2.777778N: TestBuilding12 | 0.3174603: TestBuilding13}
- TestBuilding12 -> repeat(X) {0.7043651: TestBuilding13 | 0.7440476: TestBuilding14}
- TestBuilding11 -> split(X) {2.728175N: TestBuilding15 | 0.109127: TestBuilding13}
- TestBuilding15 -> repeat(X) {0.1289683: TestBuilding13 | 0.7440476: TestBuilding14}
```
In this case the grammar gets more complex and tricky to wrap your ahead around, mostly due to the vague names of each region. But if you take the time to consider each step you will find that it describes the facade quite nicely.

The resulting building:


![Inverse Result](/citygen/images/inverseResult.png)

Do not that there are as of yet no protrude or replace rules making the surface completely flat. Furthmore the input facade image is not optimal as it contains a car on the bottom level. A better facade image would be desired and lead to a better result.

# Street Generation

The street generation starts with big rectangles that will resemble blocks. These will be cut up into the starting shapes for buildings. A texture of asphalt is added as well as a few props. An example of initial results below.

![Initial Street](/citygen/images/Street.png)

# Difficulties

```markdown
- Taking good pictures without cars and trees
- Deciding when repeat rules should be invoked
- Handling a cylinder when splitting
- Realism, mostly due to details such as signs, different cars, damages, dirt
- Perspective correction (related to difficulty of capturing image)
- Texture extraction of fitting areas for materials
- Easy way to define depth 
- Generating roofs
- Get correct material area, easier to fix manually
```
# Possible improvements

```markdown
- Other way of input rather than colors to define material. Redundant to give multiple colors for same material type to ease generation.
```
