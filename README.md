82225 Note, in this version game manager is broken. The full system is not enacted and most of the codebase for this sim is not real. The purpose is to visualize what it would look like under my theory. It's pretty damn close visually. 
82325 Note, TSP sim solves both static and dynamic pathing between 'cities' when you turn on the performance mode it will fit itself to best avg the score vs. performance or balance. 
82325 Note, Updated TSP impoved graphics, updated logic to be more honest to theory. 


TSPSIM Goal:

Dynamic = Best lowest avg means the model is fitted. You will never, or should never, get 0. The longer you run the sim the better it should be. (Except FPS cuz that's not actually a part of the logic, just the viewer)
Static aka Dynamic off = borning, don't do it. Well you can but after it solves the best route I cannot see it doing much in order to improve. It should batch the route so performance would improve but so little is thrown at it that it won't be recordable.
After your run the sim for a few minutes you need to reset your optimizer and solver for realistic goals. At the start of the sim the cluster of 'cities' are spawned very close and skew data (possible fix later, maybe)


TSPSIM bugs:

Do not increase cities. It will break. I don't care to fix it right now. It's cool as is. 
Rendering of the nodes in space probably chugging. However, it impacts the logic very little. 
This is supposed to run exclusively on the GPU, but I want to show a poc before converting this over to compute shaders. 




# Core ideas

* **BNP (Bitonic neural Precision)**: Discipline that restricts writes and precision; Q/P are read-mostly coordinates, **S** is privileged (semantic) and the only channel routinely written by logic.
* **RGBAQP+S**: Per-pixel channels: RGB (visual), Q/P (axial coords), S (semantics). Q/P guide, S encodes meaning/payload.
* **Quantized writes**: All writes are snapped to 8-bit steps (0–255) to honor BNP’s bounded precision.
* **Deterministic RNG**: XorShift32 seeded from the payload; guarantees reproducible runs per tick/tile.

# Grid & tiling

* **W×H**: Texture resolution (64×64).
* **TILE**: Tile size (8 px); **TX×TY**: tiles per axis; **NT**: total tiles.
* **idx / tIdx**: Helpers mapping (x,y)→pixel index / (tx,ty)→tile index.

# Fields & maps

* **Rails (X/Y/Z)**: Sparse binary per-pixel “tracks” (values 1,2,3); flip events toggle them.
* **RailField**: Smoothed, normalized 3-lane density (convolution radius **R**).
* **EdgeMap**: Gradient magnitude from Q/P/S; used as event bias.
* **Orientation map**: Local axis hint (≈X, ≈Y, or ≈isotropic).

# Domains & gating

* **GateTile**: Open/closed mask per tile; closed tiles reject writes.
* **DomainMask**: Per-tile permission bits: RGB | QP | S | Rails; used to clamp/allow writes.
* **applyDomain**: Enforces Gate/Domain rules on each attempted write.

# Proper-time & interpreter

* **cτ (proper-time)**: Per-tile scalar controlling micro-step depth.
* **BNP Micro-Interpreter**: Tiny VM that averages Q/P/S, softmaxes, mixes, and writes **S** only; opcodes: `NOP, LOAD_TILE, MAD, SOFTMAX3, REDUCE3, WRITE_S, JNZ, HALT`.
* **regEnergy**: Aggregate register energy per tile (telemetry).

# Eventing (DNA bus)

* **DNACompound**: 16-byte tokens pushed on “Q2” bus (class, parity, bind key, gate bits).
* **Q2\_CLASS**: Event codes (e.g., `Flip, Req, Ack, Hs, TSPImprove, TopK, AGPick, CRCok/bad, cTau, Resid`).
* **Morton key**: Z-order index used as a spatial bind key.

# Scheduler

* **bitBias**: Per-tile priority (rails + coherence + edges).
* **Bitonic proof overlay**: Visual of the bitonic sort network.
* **tileActive**: Tiles selected this tick for compute.

# NNM modules (neural-ish microsteps)

* **nnm1\_gate**: Rail-weighted RGB gain (local contrast along lanes).
* **nnm2\_pixelDrift**: Rail-aware blur/denoise of RGB.
* **nnm3\_tileMix**: Center-weighted tile color mixing.
* **nnm4\_domain**: Strict mode → write **S** from rail residuals; non-strict also nudges Q/P.
* **keepEnergy**: Keeps luminance near target.

# Residuals

* **residQ/P/S**: Slowly accumulated rail-derived residuals (used to promote semantics).
* **Residual promotion**: Boosts resid\* in hot tiles (high bitBias).

# Predictive model

* **PRED**: EMA velocity on Q/P/S and RailField with:

  * **λ (lambda)**: Blend weight (now vs predicted).
  * **Δt (horizon)**: Lookahead steps.
  * **α (EMA)**: Smoothing for velocities.
* **Pred MSE (rails/sem)**: Error between reality and last prediction.

# SpaceFrame

* **WORLD / MAP / TILE**: 3D embedding transforms for (Q,P,S) used by the 3D view.

# TSP solver

* **Cities / Tour**: Node set and current permutation (cycle).
* **Cost models**: `EUCLID`, `RAIL` (cheaper along rails), `SEM` (semantic smoothing), `MIX`.
* **Precision**: `AUTO` (MACRO/MESO/MICRO) adjusts sampling/jitter based on event load.
* **twoOptAttempt**: Classic 2-opt local swap move.
* **Top-K**: Samples K best candidate 2-opt moves per frame and applies the best delta<0.
* **Dynamic domain**: Cities drift slightly along RailField gradients.
* **NNM allocation**: Macro/Micro “worker” counts (auto/manual) governing solver effort.
* **stepsPerFrame (spf)**: Solver move budget per frame.

# Multi-objective evolution (AGPRC)

* **NSGA-II**: Maintains population of tours by 3 objectives: length, (1-rail affinity), (1-semantic homogeneity).
* **Front-0**: Non-dominated set (best trade-offs).
* **Crowding distance**: Diversity metric inside a front.
* **AGPick**: Event when an evolved tour improves the best.

# Protocol (REQ/ACK)

* **REQ/ACK**: Probabilistic handshake between neighbor tiles; counts edges & successes.
* **Fault drop %**: Simulated message loss.
* **trailEdges**: Short-lived lines for successful handshakes (2D/3D overlays).

# Self-optimizer

* **Objective**: `PERF` (FPS), `AVG` (tour length), `BAL` (blend).
* **ε (epsilon)**: Exploration chance; **amp**: tweak magnitude.
* **Interval**: Trial window before accept/revert.
* **Best params/score**: Running champion configuration.

# Performance governor

* **Eco mode**: Adapts workScale, 3D draw cadence, predictor cadence to hit target FPS.

# Visuals & overlays

* **Domain overlay**: Tile permission tint.
* **cτ heatmap**: Interpreter depth overlay.
* **Scheduler overlay**: Activity intensity per tile.
* **Predicted rails overlay**: Transparency where prediction expects rail activity.
* **TSP history band**: Strip chart of recent tour lengths.

# Misc/utility

* **Meaningful dataset**: Tri-cluster RGB with payload encoded into **S**.
* **Payload**: User text embedded in **S** (affects seeding).
* **CRC demo**: Small **S** patch tested via CRC32; emits CRCok/CRCbad events.
* **Trails (rayEdges)**: Lines drawn between tiles touched by TSP swaps.
* **Edge bias**: Scales flip probability by local edge strength.
* **Radius R**: Kernel radius for RailField smoothing.

