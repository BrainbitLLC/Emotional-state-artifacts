# Algorithm of emotional states

### Overview
The library is designed to process the signal from BrainBit headband. It allows you to get the emotional state of a person (mental levels) - in terms of the degree of relaxation or concentration, as well as the rhythms of the brain - in terms of relative expression levels for Delta, Theta, Alpha, Beta, Gamma bands. The library provides adjustable artifacts detection and elimination techniques, and signal quality analysis. It shows the presence of artifacts in EEG signal and average estimation of the signal quality. With its help you can understand how much the signal is suitable for calculations. In most cases, the presence of artifacts means a poor fit of the device to the skin or eyes and muscular related movements.

### Description
The library can work with bipolar channels or with multiple channels processing modes.

The algorithm processes raw EEG data in Volts iteratively by a sliding window of a given length with a given processing frequency.

The data is added to the library in short fragments by `pushData()` method (bipolar mode) or `pushDataArr()` (multiple channels mode). There is no strict limit for the length of input arrays in these methods, the inner buffer of the library will process it according to a chosen sliding window length and processing frequency. However, it is recommended to have these arrays length <= mathLibSetting.sampling_rate / mathLibSetting.process_win_freq.

### Filtering
The default filters set is specifically designed to eliminate low frequency components (Delta and partially Theta), because in most cases for mobile neurodevices, it does not represent neural activity, but rather artifacts. In case you are interested in this components, you still can obtain it with usage of internal filters by executing method `setZeroSpectWaves(true, 1, 1, 1, 1, 1)` with coefficient 1 for Delta. 

You can also use your own filters before sending data to the library. In such cases, the internal filtering can be turned off by `useInternalFilters(false)`.


### Artifacts
There are 2 main methods for artifacts check: 
`isBothSidesArtifacted()` and `isArtifactedSequence()`.
The first one is for detection of artifacts on all channels for the current window,
and the second one is for detection of artifacts on all channels for several consecutive windows (defined by artifactDetectSetting.global_artwin_sec in seconds).

In bipolar mode - if artifacts are detected on one of the bipolar channels, the artifacts on the second bipolar channel are checked, and if there are no artifacts there, the signal processing is switched to that channel. In case of artifacts on both bipolar channels - this region is not used for estimations, the spectral and mental values are filled with the previous valid values, and the counter of consecutive artifacted windows increases. You can specify which bipolar channel will be in priority for signal processing in case, when there are no artifacts on both sides (`setPrioritySide()`).

In multiple channels mode - if artifacts are presented on the current channel, then there is a switch to the channel with the best signal quality (see Quality of the signal part). In case of artifacts on all channels - the spectral values and values of mental levels are filled with the previous valid values, and the counter of consecutive artifacted windows increases.

When the maximum number of consecutive artifact windows (artifactDetectSetting.global_artwin_sec) is reached, `isArtifactedSequence()` returns `true`, which supposes to give the user information about the need to check the position of the device. This flag is usually raised 4 sec after receiving continuous artifacts (with the default parameter of artifactDetectSetting.global_artwin_sec). If there is no need to give notification of momentary artifacts, you can use this function as the primary for artifact notifications. Otherwise, use `isBothSidesArtifacted()` to check for momentary artifacts, returning `true` for artifacts on all channels for the current window.


### Rhythms
There are two options for calculating rhythm indices, absolute and relative. Absolute outputs raw values of micro-volts of spectrum power in the specified range for each rhythm. Or as a percentage of the total power of the entire range at any given time. Relative values are taken as some variation from the initial state, which is determined during the calibration phase of the algorithm. Relative values can also be represented as power and percentage. The advantage of relative values is a more pronounced change of these indicators in the wake of a person's state. But if during the calibration process the person will be in one of the pronounced states, for example relaxation, it will be very difficult to achieve even higher values. As for absolute values - their change is less noticeable and can be very small.

During the calibration phase, you should sit still with your eyes open and try not to think about anything.

### Quality of the signal

The signal quality metric is based on a relation between total power of filtered EEG signal and the corresponding border parameter (artifactDetectSetting.total_pow_border). It reflects the quality of the signal in percentages and also serves as a base for artifact detection. The sensitivity of artifacts detection can be adjusted by making this border parameter to higher or lower values, with the higher values - more distorted signal will be still considered with higher quality. 

The method `getEEGQuality()` returns the quality of filtered EEG signal (current implementation supports only Bipolar mode). The metric is computed as averaging across several windows (artifactDetectSetting.num_wins_for_quality_avg), by making this parameter to lower or higher value, it’s possible to make it more or less reactive in time to changes in the signal, with lower values - the metric will be more fastly reactive.

### Brain Rhythms: Delta, Theta, Alpha, Beta, Gamma (%)

The library provides an estimation of relative expression of brain waves:
Delta [1..3] Hz, Theta [4..6] Hz, Alpha [7..13] Hz, Beta [14..24] Hz, Gamma [25..49] Hz. 
To obtain these values, firstly, FFT is performed for the current window. All FFT bins are computed as: 2 * sqrt (binval_real*binval_real + binval_imag*binval_imag). Then, bins till 50 Hz are accumulated as total power sum, and bins for each rhythm interval accumulated in corresponding variables. The relation of bins for certain brain rhythm bands to total power sum is used as values of brain waves expression levels for the current signal window. The final brain rhythms values are obtained as averaging across several signal windows (mentalAndSpectralSetting.n_sec_for_averaging) using method `readSpectralDataPercentsArr()`.


Additionally, there is an option to normalize spectral bands values by bands width - turned on by default, to turn it off - use `setSpectNormalizationByBandsWidth(false)`, and option to use specific weights coefficients [0..1] for adjustment of each band level with methods `setWeightsForSpectra()` and `setSpectNormalizationByCoeffs()`. It’s possible to use either one or another normalization.
`setZeroSpectWaves()` method allows to eliminate fully certain bands.

### Mental Levels: Attention, Relaxation (%)

There are two options for obtaining mental levels values: Relative and Instant, 
both are returned in the structure from `readMentalDataArr()` or `readAverageMentalData()` methods. Mental levels estimation is mostly based on analysis of Alpha and Beta bands expression levels relation. 

Relative values are estimated as some deviation from the initial state of the user, which is determined during the calibration time of the algorithm. Therefore, Relative mental levels become available only when the calibration procedure is completed. It is worth mentioning, that if during the calibration time the person will be in one of the pronounced states, for example, in deep relaxation or high concentration, it will be very difficult then to achieve higher Relaxation or Attention values. 

Instant values are estimated from the user state based on some short time period of several last seconds (mentalAndSpectralSetting.n_sec_for_instant_estimation). There are two modes for Instant levels estimation - Independent and Dependent. The difference is that, in case of Dependent estimation (used in the library by default) - Attention and Relaxation levels are only based on Alpha and Beta values and always in clear opposite relation, the sum of them gives 100%. Whereas, for Independent estimation another approach is used, which also accounts for Theta values, to turn on this option - use `setMentalEstimationMode(true)`.

## Getting started

To get the emotional state of a person you need to follow the following steps:

1. `Install` the package for the desired platform;
2. `Create` an instance of the library;
3. `Get data` from the BrainBit device;
4. Use the library to `process` the data;
5. `Finish` working with the library.

## Usage

### Emotional states

The estimate of emotional states (mental levels - relaxation and concentration) is available in two variants:
1. immediate assessment through alpha and beta wave intensity (and theta in the case of independent assessment).
2. relative to the baseline calibration values of alpha and beta wave intensity

In both cases, the current intensity of the waves is defined as the average for the last N windows.

The algorithm starts processing the data after the first N seconds after connecting the device and when the minimum number of points for the spectrum calculation is accumulated.
When reading spectral and mental values an array of appropriate structures (`SpectralDataPercents` and `MindData`) of length is returned, which is determined by the number of new recorded points, signal frequency and analysis frequency.

### Calibration
According to the results of calibration, the average base value of alpha and beta waves expression is determined in percent, which are further used to calculate the relative mental levels.

### Library mode
The library can operate in two modes - bipolar and multichannel. In bipolar mode only two channels are processed - left and right bipolar. In multichannel mode you can process for any number of channels using the same algorithms.

## Parameters
### Main parameters description
Structure `MathLibSettings` with fields:
1. `sampling_rate` - raw signal sampling frequency, Hz, integer value
2. `process_win_freq` - frequency of spectrum analysis and emotional levels, Hz, integer value
3. `fft_window` - spectrum calculation window length, integer value
4. `n_first_sec_skipped` - skipping the first seconds after connecting to the device, integer value
5. `bipolar_mode` - enabled bipolar mode, boolean value
6. `channels_number` - count channels for multi-channel library mode, integer value
7. `channel_for_analysis` - in case of multichannel mode: channel by default for computing spectral values and emotional levels, integer value

`channels_number` and `channel_for_analysis` are not used explicitly for bipolar mode, you can leave the default ones.

The sampling rate parameter must match the sampling frequency of the device. For the BrainBit, this is 250 Hz.

Separate parameters:
1. MentalEstimationMode - type of evaluation of instant mental levels - disabled by default, boolean value
2. SpectNormalizationByBandsWidth - spectrum normalization by bandwidth - disabled by default, boolean value

### Artifact detection parameters description
Structure `ArtifactDetectSetting` with fields:
1. `art_bord` - boundary for the long amplitude artifact, mcV, integer value
2. `allowed_percent_artpoints` - percent of allowed artifact points in the window, integer value
3. `raw_betap_limit` - boundary for spectral artifact (beta power), detection of artifacts on the spectrum occurs by checking the excess of the absolute value of the raw beta wave power, integer value
4. `total_pow_border` - boundary for spectral artifact (in case of assessment by total power) and for channels signal quality estimation, integer value
5. `global_artwin_sec` - number of seconds for an artifact sequence, the maximum number of consecutive artifact windows (on both channels) before issuing a prolonged artifact notification / device position check, integer value  
6. `spect_art_by_totalp` - assessment of spectral artifacts by total power, boolean value
7. `hanning_win_spectrum` - setting the smoothing of the spectrum calculation by Hamming, boolean value
8. `hamming_win_spectrum` - setting the smoothing of the spectrum calculation by Henning, boolean value
9. `num_wins_for_quality_avg` - number of windows for estimation of signals quality, by default = 100, which, for example, with process_win_freq=25Hz, will be equal to 4 seconds, integer value

Structure `MentalAndSpectralSetting` with fields:
1. `n_sec_for_instant_estimation` - the number of seconds to calculate the values of mental levels, integer value
2. `n_sec_for_averaging` - spectrum averaging, integer value

Separate setting is the number of windows after the artifact with the previous actual value - to smooth the switching process after artifacts (`SkipWinsAfterArtifact`).

## Initialization
### Main parameters

```java
int samplingFrequency = 250;
MathLibSetting mls = new MathLibSetting(samplingFrequency, 25, samplingFrequency * 4, 4, true, 4, 0);

ArtifactDetectSetting ads = new ArtifactDetectSetting(110, 70, 800000, 100, 4, true, true,false,125);

MentalAndSpectralSetting mss = new MentalAndSpectralSetting(2,4);

EmotionalMath tMathPtr = new EmotionalMath(mls, ads, mss);
```

### Optional parameters

```java
// setting calibration length
int calibrationLength = 6;
math.setCallibrationLength(calibrationLength);

// type of evaluation of instant mental levels
boolean independentMentalLevels = false;
math.setMentalEstimationMode(independentMentalLevels);

// number of windows after the artifact with the previous actual value
int nwinsSkipAfterArtifact = 10;
math.setSkipWinsAfterArtifact(nwinsSkipAfterArtifact);

// calculation of mental levels relative to calibration values
math.setZeroSpectWaves(true, 0, 1, 1, 1, 0);

// spectrum normalization by bandwidth
math.setSpectNormalizationByBandsWidth(true);
```

## Types
#### RawChannels
Structure contains left and right bipolar values to bipolar library mode with fields:
1. LeftBipolar - left bipolar value, double value
2. RightBipolar - right bipolar value, double value

#### RawChannelsArray
Structure contains array of values of channels with field:
1. channels - double array

#### MindData
Mental levels. Struct with fields:
1. Rel_Attention - relative attention value
2. Rel_Relaxation - relative relaxation value
3. Inst_Attention - instantiate attention value
4. Inst_Relaxation - instantiate relaxation value

#### SpectralDataPercents
Relative spectral values. Struct with double fields:
1. Delta
2. Theta
3. Alpha
4. Beta
5. Gamma

#### SideType
Side of current artufact. Enum with values:
1. LEFT
2. RIGHT
3. NONE

## Usage
1. If you need calibration start calibration right after library init:

```java
math.startCalibration();
``` 

The calibration can be started at any time by executing method `startCalibration()`. Without artifacts it will last for a certain time period (by default 8 seconds), this time can be adjusted by `setCallibrationLength()` method. The calibration may last longer, if the signal will have artifacts (on all channels), it will continue while the algorithm collects enough clear signal parts for specified time. There are methods to get information about the status of calibration procedure - the progress in percentage to finish - `getCallibrationPercents()`, and end of the calibration - `calibrationFinished()`. The calibration can be performed only once for the whole session of work with the library.

2. Adding and process data
In bipolar mode:

```java
RawChannels[] samples = new RawChannels[SAMPLES_COUNT];
math.pushMonopolars(samples);
math.processDataArr();
``` 
In multy-channel mode:

```java
var samples = new RawChannelsArray[SAMPLES_COUNT];
math.pushBipolars(samples);
math.processDataArr();
``` 
2. Then check calibration status if you need to calibrate values:

```java
boolean calibrationFinished = math.calibrationFinished();
// and calibration progress
int calibrationProgress = math.getCallibrationPercents();
``` 

The calibration procedure is needed to determine the baseline user state, which further will be used to estimate Relative mental levels. During the calibration phase, the user should sit still with eyes open and try to be in a neutral mental state, while the algorithm processes signals and estimates its spectral values. At this time baseline Alpha and Beta expression levels (%) are determined, and these values will be further used for computation of Relative Attention and Relaxation metrics. Baseline Alpha and Beta levels can be obtained by method `readCalibrationVals()`.

3. If calibration finished (or you don't need to calibrate) read output values:

```java
// Reading mental levels in percent
MindData[] mentalData = math.readMentalDataArr();

// Reading relative spectral values in percent
SpectralDataPercents[] spData = math.readSpectralDataPercentsArr();
``` 

The library will return new spectral and mental values, when there are enough new points added for the next sliding window shift. The sliding window shift is an internal parameter, defined as mathLibSetting.sampling_rate / mathLibSetting.process_win_freq. When the length of input arrays in PushData methods is twice or more higher than sliding window shift value - it will return an array with several consecutive instances of spectral and mental values corresponding to several signal parts.

4. Check artifacts
4.1. During calibration

```java
if(math.isBothSidesArtifacted()){
    // signal corruption
}
``` 
4.2. After (without) calibration

```java
if(math.isArtifactedSequence()){
    // signal corruption
}
``` 

## Finishing work with the library

```java
math = null;
// native finalizer is run
``` 