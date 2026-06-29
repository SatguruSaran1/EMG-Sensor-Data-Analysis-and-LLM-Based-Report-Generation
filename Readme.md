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

Cell 1:

This cell sets up the analysis environment and the main configuration values used throughout the notebook.It also sets the plotting style, defines the paths to the task, baseline, and MVC CSV files, creates the output directory if it does not already exist, and stores the key analysis parameters such as the fatigue window size, RMS window size, band-pass cutoff frequencies, and the first 30 seconds used later as the fatigue reference period.

Cell 2:
The SENSOR_MAP dictionary tells the notebook which body side, region, and muscle each sensor belongs to, so the plots and summary tables can use anatomical names instead of raw channel numbers.

Cell 3:

This cell reads the custom Trigno CSV header and loads only the EMG columns needed for the analysis. The read_trigno_metadata() function reads the first 8 rows of the file because those rows contain Trigno metadata such as sensor names, sampling rates, and sample intervals rather than signal data. It then scans the sensor-name row, extracts each sensor ID, reads the matching sample-rate and sample-interval text from the later header rows, and stores all of that in a metadata dictionary. The load_emg_csv() function uses that metadata to load only the relevant time and EMG columns, renames them into a clean format such as t_22 and emg_22, and converts them into numeric arrays.

Cell 4:

This cell defines the signal-processing helpers used by both the fatigue and %MVC parts of the notebook. The bandpass_emg() function removes low-frequency movement artifact and high-frequency noise using a Butterworth band-pass filter, while also handling missing values and very short signals safely. window_starts() generates the start indices for sliding windows, and windowed_rms() uses those starts to calculate RMS values over overlapping time windows. The median_frequency_from_psd() and mean_frequency_from_psd() functions compute MDF and MNF from a power spectral density, and sliding_frequency_features() combines all of this by computing MDF, MNF, and RMS for each fatigue window.

Cell 5:

This cell loads the four recordings that the notebook needs: the task file, the baseline file, the left MVC file, and the right MVC file. It first creates the list of sensor IDs from the sensor map, then calls load_emg_csv() for each file so that both the dataframe and the parsed metadata are available for later calculations.

Cell 6:

This cell computes the fatigue features for the task recording and creates summary tables for each sensor. The compute_fatigue_table() function loops through every sensor, prepares the filtered signal, calculates sliding-window MDF, MNF, and RMS values, and then normalizes MDF and MNF using the first 30 seconds of the task as the reference period.The slope_per_minute() helper fits a straight line to a selected feature versus time in minutes, so the notebook can report whether MDF or MNF is rising or falling over time.
Cell 7:

This cell creates the fatigue plots. The smooth_series() helper uses a centered rolling mean so the trend lines are easier to read, and add_linear_trend() draws a dashed best-fit line on a plot when there are enough valid points. plot_overall_mdf_mnf_fatigue() combines all sensors, averages MDF and MNF by time bin, smooths the results, and plots the overall fatigue trend. plot_group_fatigue() does the same thing but separately for each body group so differences between shoulders and forearms can be seen.

Cell 8:

This cell computes the baseline RMS, the MVC reference values, and the task %MVC time series. The compute_baseline_rms() function processes the baseline recording for each sensor and stores the median RMS as the resting reference level. The mvc_source_for_sensor() function chooses the correct MVC file based on whether the sensor is on the left or right side. The compute_mvc_reference_table() function then calculates the maximum RMS during the MVC recording, subtracts the baseline RMS, and stores the corrected MVC reference for each sensor. After that, compute_percent_mvc_table() processes the task recording, computes task RMS in sliding windows, subtracts the baseline RMS, divides by the corrected MVC reference.
Cell 9:

This cell generates the %MVC plots. The plot_group_percent_mvc() function groups the task %MVC values by body region, smooths the trend over time, plots one line per group, and adds a 100% reference line so the viewer can see when the task activation is approaching or exceeding MVC.

Cell 10 (True analysis-window fatigue interpretation tables):

This cell generates interpretation tables for the sliding-window analysis and calculates fatigue and activation states. Here is a breakdown of the methodology:

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