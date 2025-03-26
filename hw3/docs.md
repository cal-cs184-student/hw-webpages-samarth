## Part 1

### Task 1

Normalized co-ordinates by dividing by image dimensions, mapped to points on camera's sensor using FOV angles. Next, created ray from camera origin through the calculated sensor point, and transformed this ray's direction from camera space to world space — most of the work was in doing the transformations right!

### Task 2
Check each ray, and only keep intersections between min_t and max_t. Find the closest hit and calculate the properties needed to update the intersection object.


### Task 3
I implemented the Möller-Trumbore algorithm for triangle intersection, as shown in lecture. Calculate edges, and helper vectors, then check for parallel rays, and find barycentric co-ordinates to locate the intersection inside the triangle. COmpute distance to see if it's within the ray's bounds, and then combine the normal to interpiolate (using barycentric co-ords)

Images: CBempty, CBspheres


## Part 2

I implemented a recursive top-down approach for BVH construction that efficiently partitions 3D space to minimize ray-primitive intersection tests. For each node, I first compute a bounding box encompassing all primitives. If there are fewer primitives than max_leaf_size, I create a leaf node. Otherwise, I need to split the collection.
For splitting, I chose the spatial median heuristic, which selects the axis with the largest extent (where the bounding box is widest) and places the splitting plane at the midpoint along this axis. This approach divides space evenly, regardless of primitive distribution. I partition primitives based on whether their centroids fall before or after this split point using std::partition for efficiency.
One important detail is handling the case where all primitives end up on one side of the split, which would cause infinite recursion. When this happens, I fall back to an object median approach, simply dividing the array of primitives in half.


The implementation of the BVH acceleration structure demonstrates dramatic performance improvements across various tests. In the cow model (5,856 primitives), rendering time decreased from 3.23 seconds without acceleration to 0.04 seconds with BVH — a 80× speedup. The impact was even more pronounced with the MaxPlanck model (50,801 primitives), where rendering time was reduced from 29.14 seconds to merely 0.05 seconds, representing a staggering 583× speedup. This efficiency is clearly reflected in the intersection tests per ray metric, which dropped from 615.8 to 5.1 for the cow model and from 4,472.3 to 6.0 for the MaxPlanck model. These results confirm that the BVH implementation effectively prunes unnecessary intersection tests by quickly eliminating large portions of geometry that don't intersect with a given ray, with efficiency gains scaling impressively as scene complexity increases.

Images: CBlucy

## Part 3

For the hemisphere sampling implementation in estimate_direct_lighting_hemisphere, I established a local coordinate system at the intersection point, with the normal aligned to the z-axis. This allowed me to work in a space where the upper hemisphere represents all possible outgoing light directions. For each sample, I randomly selected a direction in this hemisphere using our uniform hemisphere sampler. After converting this direction from local to world space, I cast a shadow ray from the hit point along this direction to see if it would intersect with any light sources. When a ray did hit a light, I evaluated the BSDF at the hit point for the sampled direction and calculated the contribution using Monte Carlo integration. Finally, I averaged these contributions across all samples to get the estimated direct lighting. 

For the importance sampling implementation in estimate_direct_lighting_importance, I directly sampled the lights in the scene. After setting up the same coordinate system, I looped through each light and sampled it according to its properties. For delta lights like point sources, I only needed one sample since all rays toward the light would be identical. For area lights, I used multiple samples to capture their extended nature. For each light sample, I used the light's sample_L method to generate a specific direction and PDF toward that light. I discarded directions where the light was behind the surface or where the PDF was near zero. After converting the world-space direction to local space, I cast a shadow ray to check if anything blocked the path between the hit point and the light. If the path was clear, I evaluated the BSDF and calculated the contribution using the Monte Carlo estimator.

Images: bunny_a.png, bunny_h.png

The importance sampling approach produces significantly less noisy results with the same number of samples because it concentrates our sampling effort where it matters most—directly on the light sources—rather than wasting samples on directions that might not contribute to the final image. The uniform hemisphere sampling creates a grainy, noisy looking image in comparison

Images: bunny_1l, bunny_4l, bunny_16l, bunny_64l

## Part 4

For indirect lighting, I implemented the at_least_one_bounce_radiance function that first computes direct lighting via one_bounce_radiance, then handles the recursive bounces. After sampling a new ray direction using the BSDF's sample_f, I create a ray from the hit point along this direction and trace it to find the next intersection. Russian Roulette with 0.7 continuation probability determines whether to terminate paths (except for the first bounce to ensure we always get at least one indirect bounce when enabled). If the path continues, I recursively call the function for the new intersection and apply the Monte Carlo estimator: BSDF × cos(θ) × indirect contribution ÷ PDF × Russian Roulette factor. The function handles both accumulation modes - either summing all bounces or returning only light from the specified max depth.

Images: bunny_1024, spheres_1024

Images: spheres_1024_indirect_only, spheres_1024_direct_only

Images: bunny_1024_m0, bunny_1024_m1, bunny_1024_m2, bunny_1024_m3, bunny_1024_m4, bunny_1024_m5


The subtle red and blue tints from the walls bleeding onto the rabbit create a more realistic appearance. This color transfer doesn't happen in traditional rasterization. The additional bounces also create more gradual and realistic shadow transitions. In general, with 2/3 bounces, the has significantly more depth and dimension. The rabbit appears more integrated into its environment.

Images: bunny_1024_m0_accum, bunny_1024_m1_accum, bunny_1024_m2_accum, bunny_1024_m3_accum, bunny_1024_m4_accum, bunny_1024_m5_accum

Accumulated bounces are generally far brigher than their unaccumulated equivalents.


Comparison of different sample rates:

Images: spheres_1024_1s_4l, spheres_1024_2s_4l, spheres_1024_4s_4l, spheres_1024_8s_4l, spheres_1024_8s_4l, spheres_1024_16s_4l, spheres_1024_64s_4l, spheres_1024_1024s_4l

## Part 5

s1 stores the sum of radiance samples, while s2 captures the sum of squared samples. After each batch of 32 samples, I assess whether a pixel has converged by calculating its mean and variance from these running sums.  For each pixel, I calculate a confidence interval which narrows as either the standard deviation decreases or the count increases. This interval, when compared against a threshold of 5% of the pixel's mean illuminance, provides a reliable convergence criteria.

When a pixel reaches this statistical threshold, sampling terminates early, allowing the renderer to divert resources to more challenging regions. Visually complex areas—reflections, refractions, receive more samples, while simpler regions like diffuse walls converge rapidly. 

Images: bunny_rate, spheres_rate