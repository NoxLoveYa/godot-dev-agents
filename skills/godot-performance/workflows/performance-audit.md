# Performance audit (owner asked)

1. **Scope and conditions.** Preset(s), resolution, the acceptance views (named cameras and world states, including the
   worst case: a dense forest walk, a storm at the shore, an underwater view), vsync off. Ask the owner before the
   windowed runs (focus) and batch them.
2. **Matrix.** One process per preset; per view: avg fps, 1% low, GPU ms avg / p99, CPU ms, draw calls, primitives.
   Write `out/perf_<tag>.txt`.
3. **Bisect the worst view** with interleaved A/B variants that remove one thing each (shadows, shadow proxies, tree
   LOD n, far layers, SDFGI, volumetric fog, sky parts, water parts, particles). Rank the deltas.
4. **Walk trace** for spikes: moving and standing still, with the top suspects off.
5. **CPU probes** headless for the top scripts.
6. **Fix list** ranked by ms saved per visual cost, each with: the change, the measured saving, the picture change
   (a before / after shot pair at the same view), the preset it applies to.
7. **Owner decides** trade-offs that change the look; write the decision, the date and the numbers into the design's
   budget table.
8. **Re-measure** the matrix with the same method after the fixes.

Report: the matrix before and after, the ranked bisect, the fixes with savings, what was left and why.
