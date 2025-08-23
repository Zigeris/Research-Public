Note, in this version game manager is broken. The full system is not enacted and most of the codebase for this sim is not real. The purpose is to visualize what it would look like under my theory. It's pretty damn close visually. 
Note, TSP sim solves both static and dynamic pathing between 'cities' when you turn on the performance mode it will fit itself to best avg the score vs. performance or balance. 



TSPSIM Goal:

Dynamic = Best lowest avg means the model is fitted. You will never, or should never, get 0. The longer you run the sim the better it should be. (Except FPS cuz that's not actually a part of the logic, just the viewer)
Static aka Dynamic off = borning, don't do it. Well you can but after it solves the best route I cannot see it doing much in order to improve. It should batch the route so performance would improve but so little is thrown at it that it won't be recordable.
After your run the sim for a few minutes you need to reset your optimizer and solver for realistic goals. At the start of the sim the cluster of 'cities' are spawned very close and skew data (possible fix later, maybe)


TSPSIM bugs:

Do not increase cities. It will break. I don't care to fix it right now. It's cool as is. 
Rendering of the nodes in space probably chugging. However, it impacts the logic very little. 
This is supposed to run exclusively on the GPU, but I want to show a poc before converting this over to compute shaders. 