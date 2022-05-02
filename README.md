# Amp-Sim
Amp simulation plugin written in eel2. Modified from original open-source plugin

## Changes Made
- Changed distortion stage 1 from hard-clipping positive samples and soft-clipping negative samples to piecewise distortion described below
- Changed distortion stage 4 from atan-clipping positive samples and soft-clipping negative samples to piecewise distortion described below
- Added additional lowpass filter after distortion stage 4

## `piecewise_dist()`
Distortion function implemented from equation 3 of paper: A Review of Digital Techniques for Modeling Vacuum-Tube Guitar Amplifiers<br>
https://direct.mit.edu/comj/article/33/2/85/94251/A-Review-of-Digital-Techniques-for-Modeling-Vacuum

```cpp
function piecewise_dist(sample)
  (
    (sample >= -1 && sample < -0.08905) ? (
        sample = -3/4*(1 - pow(1 - (abs(sample)-0.032847), 12) + 1/3*(abs(sample)-0.032847)) + 0.01;
    );

    (sample >= -.08905 && sample < 0.320018) ? (
        sample = -6.153*pow(sample, 2) + 3.9375*sample;
    );

    (sample >= 0.320018 && sample <= 1) ? (
        sample = 0.630035;
    );

    sample;
  );
```

## `processLeft()`
Analysis of the main processing function run in the @sample section

1. Input Stage Filtering
```cpp
// amp input "tilt filter": tilt (reduce frequencies) frequency spectrum around mid frequency point
// http://www.tube-tech.com/hlt-2a-a-new-approch-to-eq/
sample = amp.input.tiltL.ls.eqTick(sample)   // low-shelving filter
sample = amp.input.tiltL.hs.eqTick(sample)   // high-shelving filter
sample = amp.input.hpL.eqTick(sample)        // highpass filter
sample = amp.input.lpl.eqTick(sample)        // lowpass filter
```
2. Distortion Stage 1 - Changed from hard/soft clip to piecewise distortion function implemented from paper
```cpp
sample = piecewise_dist(sample);
```
3. Filter 1st 2 Stages
```cpp
// gain character shaping filters
sample = amp.input.eq1L.eqTick(sample)  // peak filter
sample = amp.input.eq2L.eqTick(sample)  // peak filter
```
4. Distortion Stage 2
```cpp
sample *= amp.input.dist2.gain;  // scale signal by gain factor
sample  = (samplePositive * atan(sample)) + (sampleNegative * hard(sample));  // atan-clip positive samples, hard-clip negative ones
sample *= 0.4;  // scale signal back to stay within range
```
5. Filter Stage 3
```cpp
sample = amp.input.eq3L.eqTick(sample);  // peak filter
```
6. Distortion Stage 3
```cpp
sample *= amp.input.dist3.gain;  // scale signal by gain factor
sample  = hard(sample);  // hard-clip positive & negative samples
sample *= 0.5;  // scale signal back to stay within range
```
7. Filter Stage 4
```cpp
sample = amp.input.eq4L.eqTick(sample);  // peak filter
```
8. Distortion Stage 4 - Changed from atan/soft clip to piecewise distortion function implemented from paper
```cpp
sample = piecewise_dist(sample);
amp.input.lpR.eqLP(oversampling.srate, amp.input.freq.lp, 0.5);  // lowpass filter
```
9. Filter Stage 5
```cpp
sample = amp.input.eq5L.eqTick(sample);  // peak filter
```
10. Distortion Stage 5
```cpp
sample *= amp.input.dist5.gain;  // scale signal by gain factor
sample = (samplePositive * soft(sample)) + (sampleNegative * hard(sample));  // soft-clip positive, hard-clip negative
sample *= 0.25;  // scale signal back to stay within range
```
11. Output Stage Filtering
```cpp
// revert initial tilt filtering
sample = amp.output.tiltL.ls.eqTick(sample);  // low-shelving
sample = amp.output.tiltL.hs.eqTick(sample);  // high-shelving
sample *= 0.5;
```
12. 3-Band EQ
```cpp
// configurable eq via sliders in REAPER
sample = amp.eq.band1L.eqTick(sample);  // low
sample = amp.eq.band2L.eqTick(sample);  // mid
sample = amp.eq.band3L.eqTick(sample);  // high
```
13. Resonant ~10kHz Peak
```cpp
sample = amp.output.tenKL.eqTick(sample);  // filter resonant hissing peak at 10kHz
``` 
14. Depth and Presence Dynamic Filtering
```cpp
depthSample    = depth    * envL * amp.eq.depthL.eqTick(sample)    * 0.5;
presenceSample = presence * envL * amp.eq.presenceL.eqTick(sample) * 0.5;
sample += depthSample + presenceSample;  // sum the dynamically filtered samples with the regular sample
sample *= 0.6;  // compensate for adding multiple audio signals together
```
