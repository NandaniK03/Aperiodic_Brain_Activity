
Data and Methods
------------------

 Data

The dataset utilized in this study comprises sleep records (EEG signals recorded from an entire night of undisturbed sleep) obtained from [source/acquisition method].
The dataset includes EEG data collected from 45 healthy subjects,24 meditating subjects and 21 non-meditating subjects, all falling within the age range of 19 to 74. Subjects were divided into two groups- Meditators and Controls.
For EEG recordings, the standard 10-20 electrode placement system was adopted, utilizing 19 channels (Fp1, Fp2, F3, F4, C3, C4, P3, P4, O1, O2, F7, F8, T3, T4, Fz, Cz, Pz, A1, A2) for analysis. Data was filtered in the 1-40 Hz frequency range with a sampling frequency of 500 Hz. Epoch length was set to 30 seconds.

 Methods

The EEG signals were segmented using a sliding window approach, where each window spanned 4 seconds with a 2-second step (ensuring a 50% overlap) for every channel. Number of samples in each window was four times the sampling frequency, frequency resolution was 0.25 Hz, frequency range was 0-40 Hz. Power Spectral density was computed using Welch's Periodogram.

Spectral parameters were derived using the FOOOF (fitting oscillations and one over f) algorithm. It extracts both aperiodic parameters (spectral slope and intercept) and periodic parameters(central frequency, power and bandwidth) from the neural power spectra.

FOOOF algorithm models the neural power spectra as a combination of an aperiodic component (represented by an 1/f slope) and a periodic component(characterized by oscillatory peaks). To achieve this, the algorithm initially aproximates the aperiodic components and subsequently subtracts the 1/f slope from the original spectra. This results into a flattened spectra with exposed peaks.
Next, the algorithm iteratively fits and subtracts Guassian functions from the flattened spectra, removing the identified peaks. The aperiodic component is refitted. Finally, the FOOOF algorithm combines the aperiodic and periodic components.