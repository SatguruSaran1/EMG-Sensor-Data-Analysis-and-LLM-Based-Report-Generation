1. Load the raw Trigno CSV files.
2. Read the custom Trigno header to locate sensor columns and sampling metadata.
3. Extract and filter each EMG signal.
4. Compute fatigue features using sliding-window PSD analysis:
   - MDF: Median Frequency
   - MNF: Mean Frequency
   - RMS: Root Mean Square
5. Compute baseline RMS from the resting recording.
6. Compute MVC reference RMS from the left or right dynamometer recording depending on sensor side.
7. Compute %MVC for the task recording using baseline-corrected RMS.
8. Generate summary tables and plots.
9. Save all derived CSV files and figures to the output folder.

## Input files
`Avnish_push1_01.csv`  
Main task recording used for fatigue and %MVC analysis.
`Avnish_push1_baselinerest.csv`  
Resting baseline recording used to estimate baseline EMG activity.
`Avnish_push1_leftdynamo_01.csv`  
Left-side MVC reference recording.
`Avnish_push1_righydynamo_01.csv`  
Right-side MVC reference recording.

## The following main settings
 Band-pass filter: 20–450 Hz
 Fatigue window: 4 seconds
 Fatigue step: 2 seconds
 RMS window: 1 second
 RMS step: 0.25 seconds
 Fatigue reference period: first 30 seconds of the task recording

## The analysis has two primary goals:

Muscle Fatigue Analysis
Uses frequency-domain EMG features:
Median Frequency (MDF)
Mean Frequency (MNF)
Tracks how these features change throughout the task.
A decrease in MDF/MNF is commonly associated with muscle fatigue.
Muscle Activation Analysis (%MVC)
Uses RMS amplitude.
Compares task activation against MVC recordings.
Reports activation as a percentage of maximum voluntary contraction (%MVC).



### Why I choose windows the way I did
I used sliding (overlapping) windows rather than analyzing the entire signal as a single block to track dynamic changes over time.
- **Fatigue (MDF/MNF):** A 4-second window with a 2-second step. This balances the need for enough samples to get a reliable Power Spectral Density (PSD) resolution against the need to track changes over time. Source Paper: https://zenodo.org/records/14182446?preview_file=code.ipynb
Calculating Median/Mean Frequency requires converting the signal from the time domain to the frequency domain using a Fast Fourier Transform (FFT) or Welch's method. To get high frequency resolution (narrow frequency bins), you must have a large number of samples. 

- **Activation (RMS):** A 1-second window with a 0.25-second step. Amplitude changes more rapidly, so a shorter window provides finer temporal resolution to capture quick bursts of muscle activation.RMS operates in the time domain and measures the actual mechanical activation/recruitment of the muscle. Muscle activation changes very rapidly hence the window cannot be the same as mdf/mnf.
https://www.sciencedirect.com/science/article/pii/S1356689X14001210

### Normalization Process
To make the raw values comparable across different sensors and subjects, we normalize them:
- **MDF/MNF (Fatigue):** Normalized as a percentage of the initial baseline median value (taken from the first 30 seconds of the task recording). A drop below 100% indicates a shift toward lower frequencies, which is characteristic of localized muscle fatigue.
- **RMS (Activation):** First, the resting baseline RMS is subtracted from the task RMS. This corrected value is then divided by the baseline-corrected Maximum Voluntary Contraction (MVC) RMS. This yields the **%MVC**, representing how hard the muscle is working relative to its maximum capacity.

### Computation within the Moving Window
- Overlapping time segments are extracted using the predefined window size and step.
- For **RMS**, we calculate the square root of the mean of the squared signal values within each sliding window.
- For **MDF/MNF**, we estimate the Power Spectral Density (using Welch's method) for the windowed signal and then extract the median and mean frequencies.

### Calculation of Trends (Slopes and IQR)
To determine if a muscle is progressively fatiguing or recovering, we look at the trend over a recent lookback period (e.g., a rolling 60-second window):
- **Slope:** A linear regression is applied to the normalized values within the rolling 60-second window to find the slope, which represents the rate of change (e.g., percentage points per minute). A negative slope for MDF/MNF strongly suggests active fatiguing.
- **IQR (Interquartile Range):** Calculated as the 75th percentile minus the 25th percentile of the values in the rolling window. This provides a robust measure of variability, showing how noisy or stable the signal features are during that minute.
