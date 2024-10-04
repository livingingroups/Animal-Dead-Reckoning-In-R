
# Gundog.Tracks: Dead-Reckoning Animal Movement Function

## Overview

**Gundog.Tracks** is an R function designed for dead-reckoning animal movement with an optional feature for Verified Position Correction (VPC). It generates a corrected trajectory for animal movements using input parameters such as heading, velocity, external current interactions, and optionally altitude/depth.

The function also provides options for **visualizing the movement** by plotting the corrected tracks.

## Features

- **Dead-Reckoning**: Estimate animal positions over time based on movement sensors (heading and speed estimates derived from tri-axial ACC and MAG data).
- **Verified Position Correction (VPC)**: Correct estimated positions with known positions if available, with several methods for under-sampling Verified Positions.
- **Customizable Plotting**: Visualize tracks if `plot = TRUE`.
- **Handles Various Input Types**: Altitude/Depth, compass headings, speed/DBA, and other metrics can be provided.
- **Directional Movement**: Function allows dead-reckoning in both forward (`Outgoing = TRUE`) and reverse directions (`Outgoing = FALSE`).

## Installation

To use this function, ensure you have **R** installed on your system. The function requires several R packages that are commonly used for data manipulation, visualization, and spatial analysis.

### Required Libraries

The following R packages are required:
- **zoo**: Used for working with time series data.
- **dplyr**: Used for data manipulation.
- **ggplot2** (optional, for plotting): Used for creating plots.
- **plotly** (optional, for interactive plotting): Used if `interactive.plot = TRUE`.

### Automatic Package Installation

The app will automatically install any missing packages upon startup. If a required package is not installed, the script will install it before proceeding. You can review the package installation section in the code:

```r
# Required packages
if (!require('zoo')) { install.packages('zoo', dependencies = TRUE, type="source") }
suppressMessages(require("shiny"))
# (and so on for other packages...)
```

### Dependencies Notes
- **Preferably detach `imputeTS` and `plyr` packages** before using this function, as they might conflict with `zoo` and `dplyr`.

## Function Usage (For additional details, please refer to the comments within the script.).

### Function Signature

```r
Gundog.Tracks = function(
  TS, h, v, elv = 0, p = NULL, cs = NULL, ch = NULL, m = 1, c = 0, max.speed = NULL, ME = 1, lo = 0, la = 0,
  VP.lon = NULL, VP.lat = NULL, VP.ME = FALSE, method = NULL, thresh.t = 0, thresh.d = 0, dist.step = 1, Dist_Head.corr = FALSE,
  bound = TRUE, Outgoing = TRUE, plot = FALSE, plot.sampling = 0, interactive.plot = FALSE
)
```

### Parameters

- **TS**: Timestamp vector (as `POSIXct` object). Ensure decimal seconds are expressed for data obtained at > 1Hz.
- **h**: Magnetic heading data (in degrees [0-360°]).
- **v**: Input speed (m/s) or dynamic body acceleration (DBA) metric of choice (e.g., VeDBA).
- **elv** (optional, default = 0): Elevation/depth data (meters) for 3D distance estimates.
- **p** (optional): Pitch data, used to adjust radial distance (modulated according to pitch).
- **cs** (optional): Current speed (m/s), either as a single value or vector. Missing values (NAs) are replaced with previous values.
- **ch** (optional): Current heading (in degrees [0-360°]), either as a single value or vector. Missing values (NAs) are replaced with previous values.
- **m** (optional, default = 1): Multiplier to adjust speed (e.g., gradient). This is typically used when supplying DBA to scale estimates linearly up to speed values in m/s. Can be supplied as a single value or a vector. Missing values (NAs) are replaced with the most recent non-NA value.
- **c** (optional, default = 0): Constant to adjust speed (e.g., intercept).This is typically used when supplying DBA (Dynamic Body Acceleration) to scale estimates linearly up to speed values in m/s. Can be supplied as a single value or a vector. Missing values (NAs) are replaced with the most recent non-NA value.
- **max.speed** (optional): Cap for speed values (m/s). Note that this cap applies before Verified Position Correction (VPC). Post-VPC, dead-reckoned (DR) speed estimates may exceed the capped speed, depending on the amount of track segment expansion required to align with their endpoint VPs.
- **ME** (optional, default = 1): Marked Events indicating movement. Values greater than 0 indicate movement, while 0 indicates stationary. When ME = 0, tracks will not advance, even if input speed/DBA is high, and these sections of data will not be expanded during the VPC procedure.
- **lo**, **la**: Initial longitude and latitude coordinates (in decimal format).
- **VP.lon**, **VP.lat** (optional): Verified Positions (longitude and latitude) synced to movement data, can contain NAs.
- **VP.ME** (optional, default = FALSE): If `TRUE`, verified positions are disregarded when `ME = 0` (animal is stationary).
- **method** (optional): Method to under-sample VPs for VPC.

## Method Parameter: Detailed Description

The **method** parameter defines how the Verified Positions (VPs) are under-sampled before the Verified Position Correction (VPC) procedure in dead-reckoning. There are five options available: `'All'`, `'Divide'`, `'Time_Dist'`, `'Cum.Dist'`, and `'Time_Dist_Corr.Fac'`. If the **method** parameter is not provided, the function defaults to `NULL`, implying that no VPC will be applied.

### VP.ME Considerations
If **VP.ME** is set to `TRUE`, only VPs that occur when **Marked Events (ME)** are greater than `0` are used for VPC from that point onwards. The initial coordinates (`lo` and `la`) are not filtered out even if **ME** is `0` and **VP.ME** is set to `TRUE`.

### Method Options:

1. **Divide**:
   - The **Divide** method under-samples VPs by splitting the VP data frame into equal segments based on row numbers. The first and last fixes are always retained.
   - The segmentation is adjusted depending on the number of rows between the first and last fix, and it is based on the value of **thresh.t** provided by the user.
     - For example, if **thresh.t** is set to `1`, there is only one correction applied. If **thresh.t** is `2`, two corrections are applied (i.e., the first, middle, and last coordinates are used).
   - This method is only recommended when there is a high success rate for obtaining VPs, as missing data can lead to uneven correction intervals over time.

2. **Time_Dist**:
   - The **Time_Dist** method under-samples VPs based on accumulated **time** (in seconds) between fixes, the straight-line **distance** (in meters) between VPs, or a combination of both.
   - If **thresh.t** is provided and greater than `0`, the function retains a fix whenever the accumulated time between VPs reaches the threshold. This process continues iteratively throughout the dataset. 
     - For example, if GPS data is recorded at `1 Hz` and **thresh.t** is `60`, a fix is retained every minute, assuming no missing GPS data.
   - If **thresh.d** is provided and greater than `0`, the function retains a fix whenever the straight-line distance (using the Haversine formula) between the last retained fix and each candidate fix reaches the threshold. This process also continues iteratively.
   - The input **dist.step** determines the stepping interval for calculating distance between VPs. For instance, **dist.step = 5** means the distance is calculated every 5th VP, regardless of the time between them, in a rolling manner.
   - If both **thresh.t** and **thresh.d** are provided, then both thresholds must be met before a subsequent fix is retained.
   - The first and last fixes are always retained as default.

3. **Cum.Dist**:
   - The **Cum.Dist** method under-samples VPs based on accumulated distance (in meters) between VPs (not using straight-line distance).
   - It calculates the maximum accumulated 2-D distance between VPs for a given stepping range, which is determined by **dist.step**.

4. **All**:
   - The **All** method retains every VP for use in the VPC procedure, regardless of thresholds set.
   - If **VP.ME** is `TRUE`, VPs occurring during **ME = 0** (when the animal is stationary) are disregarded before this step.
   - This method is not recommended for high-resolution VP datasets (e.g., GPS recorded at `1 Hz`) as it may lead to inefficiencies and over-correction.

5. **Time_Dist_Corr.Fac**:
   - The **Time_Dist_Corr.Fac** method is an experimental approach intended to improve the accuracy of VPC-dead-reckoned tracks, particularly when speed and heading estimates are inaccurate or when high-resolution VP data are noisy.
   - This method uses **thresh.t** and **thresh.d** inputs similar to the **Time_Dist** method. It goes further by evaluating correction factors over a **span** period (in seconds) beyond the candidate fix to retain, and then selects the fix with the lowest associated magnitude of correction factor(s).
   - If **Dist_Head.corr** is set to `FALSE`, only the distance correction factors are considered when selecting a fix. The process continues iteratively from this retained VP.
   - If **Dist_Head.corr** is `TRUE`, both the distance and heading correction factors are evaluated. Since these two metrics have different scales, both are normalized and summed before selecting the fix with the lowest summed normalized correction factor. This process also continues iteratively.

### Additional Parameters:

- **dist.step**: Defines the stepping interval used for calculating distances between VPs, both for summarizing VP distances and for under-sampling methods that use distance. For example, **dist.step = 5** means distances are calculated every 5th VP in a rolling fashion.

- **span**: Specifies the period (in seconds) used within the **Time_Dist_Corr.Fac** method to evaluate correction factors for all intermediate VPs within the window. For instance, **span = 180** examines correction factors of all VPs within a three-minute window from the current candidate VP.

- **bound**:
  - If **bound = TRUE** (default), the VPC dead-reckoning process will be bounded by the first and last VP present. This means that the dead-reckoned track will begin at the specified initial coordinates (`lo`, `la`) and end at the last VP, following any **VP.ME** filtering if enabled.
  - If **bound = FALSE**, the dead-reckoning track will extend beyond the last VP and continue until the end of the movement sensor data. The uncorrected segment will inherit the last available correction factors from the preceding corrected segment.

- **Outgoing** (optional, default = TRUE): If `FALSE`, performs dead-reckoning in reverse. This is useful when the pre-determined position is unknown, but the finishing coordinates are.


- **plot** (optional, default = FALSE): If `TRUE`, generates summary plots.

### Standard Dead-Reckoning (Without VPC)
The user can opt for standard dead-reckoning without applying Verified Position Correction (VPC) by leaving the `VP.lat`, `VP.lon`, and `method` fields blank in the function input (the default is `NULL`). In this case, the following parameters are ignored: `VP.ME`, `bound`, `thresh.t`, `thresh.d`, `method`, `span`, and `Time_Dist_Corr.Fac`. This allows for straightforward dead-reckoning without involving any corrections from verified positions.

### Plotting Options
If `plot = TRUE` and VPC is enabled, a series of summary plots will be generated to visualize the relationship between the dead-reckoned (DR) track and the verified position (VP) track, along with error estimates. Specifically, the following are displayed:
- **Two levels of VPC comparison**:
  1. **No VPC**: Displays the DR track before and after current integration (if current information is supplied).
  2. **Corrected DR Track**: Displays the DR track with VPC applied based on the under-sampling method used.
- **Uncorrected vs. Corrected DR Tracks**: Both uncorrected and corrected DR tracks are plotted to facilitate comparison.

If `plot = TRUE` without VPC, only the original DR track will be plotted, providing a basic visualization of the dead-reckoning path.

### Plot Sampling
The `plot.sampling` parameter allows the user to specify the level of under-sampling for plotting purposes, with units in seconds. This can be particularly useful for large datasets, as it speeds up the plotting process by reducing the number of points plotted. For example, setting `plot.sampling = 60` retains one value every 60 seconds.

### Interactive Plotting
If `interactive.plot = TRUE`, an interactive version of the VPC dead-reckoned track will be plotted using the `ggplotly` library (default is `FALSE`). This requires the `ggplot2` and `plotly` packages.

**Warning**: Avoid using `interactive.plot = TRUE` with very large datasets, as this may cause performance issues. It is recommended to under-sample infra-second data using `plot.sampling` when working with larger datasets.

When `interactive.plot = TRUE`, the function will return a list containing both the interactive plot (`ggplotly` figure) and the data frame, instead of just the data frame.


## Outputs

The function returns a data frame with the following columns if VPC is initialized (presence of some columns depends on the input parameters):

1. **Row.number**: Row number of the output.
2. **Timestamp**: Timestamp for each observation.
3. **DR.seconds**: Accumulated time difference between rows (in seconds).
4. **Heading**: Original heading data.
5. **Marked.events**: Original marked events.
6. **DBA.or.speed**: Original speed or dynamic body acceleration.
7. **Pitch** (if provided): Original pitch data.
8. **Radial.distance**: Calculated q coefficient (prior to VPC factors).
9. **Elevation** (if provided): Original elevation data.
10. **Elevation.diff** (if provided): Rate of change of supplied elevation/depth (in m/s).
11. **Current.speed** (if provided): Original current speed.
12. **Current.heading** (if provided): Original current heading.
13. **Heading.current.integrated** (if provided): Updated heading after integrating current vectors (prior to VPC).
14. **Radial.distance.current.integrated** (if provided): Updated q coefficient after integrating current vectors (prior to VPC).
15. **DR.longitude**: Original dead-reckoned longitude without VPC (possibly with currents integrated).
16. **DR.latitude**: Original dead-reckoned latitude without VPC (possibly with currents integrated).
17. **DR.longitude.corr** (if provided): Corrected dead-reckoned longitude coordinates (post VPC).
18. **DR.latitude.corr** (if provided): Corrected dead-reckoned latitude coordinates (post VPC).
19. **Dist.corr.factor** (if provided): Distance correction factor (observations carried forward).
20. **Head.corr.factor** (if provided): Heading correction factor (observations carried forward).
21. **Heading.corr** (if provided): Corrected heading.
22. **Radial.distance.corr** (if provided): Corrected q coefficient.
23. **Distance.error.before.correction** (if provided): Distance (in meters) between uncorrected dead-reckoned positions and VPs (observations carried forward).
24. **Distance.error.after.correction** (if provided): Distance (in meters) between corrected dead-reckoned positions and VPs (observations carried forward).
25. **DR.distance.2D**: Two-dimensional distance moved (in meters) between dead-reckoned fixes.
26. **DR.distance.3D** (if provided): Three-dimensional distance moved (in meters) between dead-reckoned fixes.
27. **DR.cumulative.distance.2D**: Accumulated two-dimensional distance moved (in meters) between dead-reckoned fixes.
28. **DR.cumulative.distance.3D** (if provided): Accumulated three-dimensional distance moved (in meters) between dead-reckoned fixes.
29. **DR.distance.from.start.2D**: Two-dimensional (straight-line) distance moved (in meters) from starting position.
30. **DR.distance.from.start.3D** (if provided): Three-dimensional (straight-line) distance moved (in meters) from the starting position.
31. **DR.speed.2D**: Horizontal speed (in m/s) calculated as `DR.distance.2D / time difference between rows`.
32. **DR.speed.3D** (if provided): Total speed (in m/s) calculated as `DR.distance.3D / time difference between rows`.
33. **VP.seconds** (if provided): Accumulated time (in seconds) between supplied VPs positions (observations carried forward).
34. **VP.longitude** (if provided): Supplied VP longitude coordinates (observations carried forward), sub-sampled if `VP.ME = TRUE`.
35. **VP.latitude** (if provided): Supplied VP latitude coordinates (observations carried forward), sub-sampled if `VP.ME = TRUE`.
36. **VP.present** (if provided): Indicates when a VP was present (`1`) or absent (`0`), sub-sampled if `VP.ME = TRUE`.
37. **VP.used.to.correct** (if provided): Indicates which fixes were used within the VPC procedure (`1 = used` or `0 = ignored`).
38. **Number.of.VPCs** (if provided): Increments by 1 each time a VP was used within the VPC procedure (observations carried forward).
39. **VP.distance.2D** (if provided): Two-dimensional distance moved (in meters) between VPs (using stepping interval 'dist.step').
40. **VP.cumulative.distance.2D** (if provided): Accumulated two-dimensional distance moved (in meters) between VPs.

### Example Usage

```r
# Example data
TS <- as.POSIXct(seq(Sys.time(), by = "min", length.out = 100))
heading <- runif(100, min = 0, max = 360)  # Random heading in degrees
velocity <- runif(100, min = 0, max = 5)   # Random speed in m/s

# Running the function with sample data
tracks <- Gundog.Tracks(TS = TS, h = heading, v = velocity, plot = TRUE)
```

### Notes

- **VPC Methods**: Several methods are provided for Verified Position Correction. The appropriate method should be chosen based on data quality and availability.
- **VP Under-Sampling**: If VPC is applied too frequently (e.g., >= 0.2 Hz), it may cause the DR track to inherit errors from VPs. It is advised to apply under-sampling to avoid these issues.
- **Movement Marked Events (`ME`)**: Using `ME` properly helps in improving the correction process and avoiding unnecessary position updates.
- **Interactive Plotting**: Use the `interactive.plot = TRUE` option only for small datasets due to computational requirements.

## License

This project is licensed under the MIT License - see the `LICENSE` file for details.

## Contact

For questions or suggestions, feel free to contact me at [rgunner@ab.mpg].

