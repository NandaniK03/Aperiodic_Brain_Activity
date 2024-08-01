
Code Script
=============

General Description
--------------------

This script aims to compute the aperiodic component from sleep data and explore the findings across different sleep stages. 

Steps:  

1. Load the sleep data and its corresponding hypnogram for all subjects. 

2. Data Cleanup:  

* Epoch the data into 30-second intervals.  

* Extract the hypnogram as a sequence for every 30 seconds to align it with the sleep data.  

3. Power Spectral Density (PSD) Computation: 

* For each channel, compute the Power Spectral Density (PSD) using Welch's method with a median of the 4-seconds a 2-seconds step (50% overlap) for each channel

4. Fit Aperiodic Components:  

* Apply the FOOOF algorithm for each 30-second epoch of the PSD data.  

* Save the results of the FOOOF analysis for the aperiodic and periodic parameters. 

5. Sleep Stage Averaging: 

* Average the aperiodic parameters across Wake, N1, N2, N3, and REM sleep stages for each channel. 

6. Data Aggregation: 

* Save the averaged aperiodic parameters to a dataframe for further analysis.  

Note:  

* Ensure that the input sleep data and hypnogram files are in the correct format and structure for proper data processing. 

* The script uses FOOOF and PSD to analyze sleep data. Please refer to the respective papers and documentation for more information.  

* Error handling and code comments are included in the script to enhance code clarity and maintainability. 



Code Description
-----------------

1. Load necessary libraries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this code block, we start by loading several essential libraries that will be used throughout the script:

* `glob`: This library has been used to load all PSG(Polysomnography) files and corresponding Scored files with .edf extension. It can retrieve the file paths.

* `os`: The `os` library provides functions for interacting with operating system. It has been used to perform file-related operations.

* `yasa`: `yasa` has been used for slicing hypnogram into 30-second windows and creating topoplots for visualization.

* `numpy`: It is a fundamental library for working with numerical data in Python.

* `matplotlib`: This library is amazing for visualizing data. It has been used to create plots and charts.

* `scipy` : welch module from `scipy` has been utilized for computing the Power Spectral Density(psd) of EEG signal. stats module has been used to take trimmed mean of data.

* `seaborn`: `seaborn` provides a high-level interface for drawing attractive and informative statistical graphs

* `mne`: It has been used for reading edf files conatining EEG data

* `pandas`: `pandas` is useful for data manipulation and analysis. 

* `FOOOF`: `FOOOF` has been used for model fitting. FOOOF is a fast, efficient, and physiologically-informed tool to parameterize neural power spectra. 


.. code-block:: python
   
   #%% Import necessary libraries | Block 0

   import glob as gb  
   import os
   import yasa
   import numpy as np
   import matplotlib.pyplot as plt
   from scipy.signal import welch
   from scipy.stats import trim_mean
   import seaborn as sb
   import mne
   import pandas as pd
   from fooof import FOOOF


2. Load files
~~~~~~~~~~~~~~~

The code block below deals with loading files, specifically PSG (Polysomnography) and scored files(hypnogram), for further processing and analysis.

* First we specify the folder path where PSG files/ Scored files are located

* using ``glob`` module. we retrieve a list of files in the specified folder with the extension ``.edf``.

* ``psg_files_all`` /  ``scored_files_all`` list is by default loaded in the lixographic order. These can be sorted in alphabetical order of filenames using ``sorted()`` function.

* List of filenames: We change the current working directory to the folder path where PSG files are located (`folder_path_psg`). Using glob, we retrieve a list of PSG files again. The ``psg_files`` list is sorted to ensure alphabetical ordering of the filenames.

.. code-block:: python

   #%% Load files | Block 1

   # PSG files

   # Retrieve a list of files in the folder with a specific extension
   # Specify the file extension or pattern and the folder path
   folder_path_psg = '/serverdata/ccshome/nandanik/Documents/FOOOF_data/data'
   file_pattern_psg = '*.edf'

   # also convert lixographic order to alphabetical order
   psg_files_all = sorted(gb.glob(folder_path_psg + '/' + file_pattern_psg))

   # Scored files

   # Retrieve a list of files in the folder with a specific extension
   # Specify the file extension or pattern and the folder path
   folder_path_scored = '/serverdata/ccshome/nandanik/Documents/FOOOF_data/metadata'
   file_pattern_scored = '*.edf'

   # also convert lixographic order to alphabetical order
   scored_files_all = sorted(gb.glob(folder_path_scored + '/' + 
   file_pattern_scored))

   # List containing files names
   os.chdir(folder_path_psg)
   psg_files = sorted(gb.glob( file_pattern_psg))


3. Load Data
~~~~~~~~~~~~~~

In this code block (Block 2), data loading and processing is performed, including retrieving the 19 required channels along with A1 and A2, filtering the data, and computing the Power Spectral Density (PSD) for each PSG file. Here's break down of the code:

**a. Data Loading:**

* Initialize empty lists ``hypno_30s_all`` to store sleep stage labels for each epoch (30s window) and ``psd_all`` to store PSDs for all PSG files.

* For each pair of PSG and scored files obtained from ``zip(psg_files_all, scored_files_all)``, we read the PSG data using ``mne.io.read_raw_edf`` and store the sampling rate (srate).

**b. Channel Selection and Bandpass Filtering:**

* 19 specific channels are to be picked, including A1 and A2, from the PSG data using MNE's ``pick_channels``.

* Note that different PSG files may contain different format of labels for electrodes. Eg. Fz electrode could eb labelled as- 'Fz', 'FZ', 'EEG Fz', 'EEG Fz'. In the ``channels_to_pick`` list, specify these labels in the order from Fp1 to A2.

* The data is bandpass filtered between 1 Hz and 40 Hz using ``edfdata.filter``.

**c. Hypnogram Cleanup:**

* Read the hypnogram annotations using MNE and convert them to a DataFrame for easier manipulation using ``mne.read_annotations`` and `` to_data_frame``.

* The onset column is converted into epoch numbers, representing the start of each epoch in the hypnogram (``timestamps, only_time, and epochs_start``).

* The description column is modified to contain only the sleep stage labels (``just_labels``).

* Clean hypnogram is created by repeating each sleep stage label for the duration of the corresponding epoch (``hypno_30s``) for each PSG file. The labels for each PSG file are added to ``hypno_30s_all`` in the form of a nested list.

**d. Power Spectra Computation:**

* The data is cut into 30-second epochs using ``yasa.sliding_window``.

* Ensure that data and hypnogram have the same shape. 

* The PSD is computed using Welch's method (``welch``) for each 30-second epoch for every PSG file.

* Frequency range is sliced from 0 Hz to 40 Hz for analysis.

* The resulting PSDs (for each PSG file) are iteratively stored in the ``psd_all`` as nested list .

By the end of this code block, we will have sleepstage labels and PSDs for each epoch corresponding to individual PSG files (`hypno_30s_all``, ``psd_all``) both stored as nested lists. 

.. code-block:: python

   #%%  Load the data | Get the 19 channels required + A1 and A2 | Block 2

   # initialize empty lists to store sleepstage labels ad psds for all PSG files
   hypno_30s_all = []
   psd_all = []

   for PSGfiles, scored_files in zip(psg_files_all, scored_files_all):
       data = []   
       hypno_30s = []

       # read psgfile
       edfdata = mne.io.read_raw_edf(PSGfiles, preload=True)
       srate = int(edfdata.info['sfreq'])

       # Get the 19 channels required + A1 and A2
       channels_to_pick = ['Fp1', 'FP1', 'Fp2', 'FP2', 'F3', 'F4', 'C3', 'C4',
                          'P3', 'P4', 'O1', 'O2','F7', 'F8', 'T3', 'T4', 'Fz', 
                           'FZ', 'Cz', 'CZ', 'Pz', 'PZ', 'A1', 'A2']

       edfdata.pick_channels(channels_to_pick)

       # bandpass filter data
       edfdata.filter(1,None,fir_design='firwin').load_data()
       edfdata.filter(None,40,fir_design='firwin').load_data()
       data = edfdata.get_data() * 1e6 #coverting volts to microvolts

   # Cleanup the hypnogram data into a sequence of stages every epoch
       hypnogram = mne.read_annotations(scored_files)
       hypnogram_annot = hypnogram.to_data_frame()

       # change the duration column into epochs count
       hypnogram_annot.duration = hypnogram_annot.duration/30

       # convert the onset column to epoch number
       timestamps = hypnogram_annot.onset.dt.strftime("%m/%d/%Y, %H:%M:%S")
       only_time = []
       for entries in timestamps:
           times = entries.split()[1]
           only_time.append(times.split(':'))

       # converting hour month and seconds as epoch number
       epochs_start = []
       for entries in only_time:
           hh = int(entries[0]) * 120
           mm = int(entries[1]) * 2
           ss = int(entries[2])/ 30

           epochs_start.append(int(hh+mm+ss))

       # replacing the onset column with start of epoch
       hypnogram_annot['onset'] = epochs_start
       epochs_start = []
       for entries in only_time:
           hh = int(entries[0]) * 120
           mm = int(entries[1]) * 2
           ss = int(entries[2])/ 30

           epochs_start.append(int(hh+mm+ss))

       # replacing the onset column with start of epoch
       hypnogram_annot['onset'] = epochs_start

       # keep the description column neat
       just_labels = []
       for entries in hypnogram_annot.description:
           just_labels.append(entries.split()[2])

       # replacing the description column with just_labels
       hypnogram_annot['description'] = just_labels

       # we need only the duration column and description column to recreate hypnogram
       # just reapeat duration times the label in description column
       # adding labels for every second of sleep data
       for stages in range(len(hypnogram_annot)):
           for repetitions in range(int(hypnogram_annot.duration[stages])):
               hypno_30s.append(hypnogram_annot.description[stages])

       # append hypno_30s for each file
       hypno_30s_all.append(hypno_30s)

   # Power Spectra

   # Generate 30 seconds PSDs for all channels
   # cut data into 30 seconds epochs

       # cutting data into 30s epochs using sliding_window() function
       _, data = yasa.sliding_window(data, srate, window=30)

       # Make sure the hypnogram is also same size as data
       # This would imply removing last part of scoring string
       hypno_30s = hypno_30s[0:np.shape(data)[0]]

       # compute power spectrum
       win = int(4 * srate)  # Window size is set to 4 seconds
       freqs, psd = welch(data,
                       srate,
                       nperseg=win,
                       noverlap= int(win*0.5), #50% overlapping
                       axis=-1)

       # Slicing the frequency ranges from 0 Hz until 40 Hz
       index_40_hz = np.where(freqs == 40)[0] + 1
       psd = psd[:,:, 1:index_40_hz[0]]
       freqs = freqs[1:index_40_hz[0]]

       # append psd for each file
       psd_all.append(psd)
	   
4. Assigning PSDs to respective sleepstages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* The power spectral density (PSD) data is organized based on their corresponding sleep stages for each subject. For each subject, we iterate through their PSD data and match it with the sleep stage labels collected earlier in ``hypno_30s_all``. 

* PSD data is organized into 5 sleepstages- W, N1, N2, N3, REM for individual subject file(``psd_w``, ``psd_n1``, ``psd_n2``, ``psd_n3``, ``psd_rem``). 

* The arrays, psd_sub_w, psd_sub_n1, psd_sub_n2, psd_sub_n3, and psd_sub_rem, hold the PSD data corresponding to each epoch, channel, and frequency range for all subjects and their respective sleep stages. 

* This organization of data facilitates efficient and structured exploration and interpretation of the results across sleep stages.

.. code-block:: python

   #%% psds correspnding to respective sleepstages | Block 3

   # initialize lists for containing psds for all subjects corresponding to 5 sleep stages
   psd_sub_n1 = []
   psd_sub_n2 = []
   psd_sub_n3 = []
   psd_sub_rem = []

   for psd,hypno_30s in zip(psd_all,hypno_30s_all):

       # initialize lists for psd for each subject corresponding to sleep stages
       psd_w = []
       psd_n1 = []
       psd_n2 = []
       psd_n3 = []
       psd_rem = []

       # index by index matching of elements in PSD and hypnos_30 for individual subject file
       for i,sleepstage in zip(psd,hypno_30s):
           if sleepstage == 'W':
               psd_w.append(i)
           elif sleepstage == 'R':
               psd_rem.append(i)
           elif sleepstage == 'N1':
               psd_n1.append(i)
           elif sleepstage == 'N2':
               psd_n2.append(i)
           elif sleepstage == 'N3':
               psd_n3.append(i)

       # append stagewise psds for each file
       psd_sub_w.append(psd_w)
       psd_sub_n1.append(psd_n1)
       psd_sub_n2.append(psd_n2)
       psd_sub_n3.append(psd_n3)
       psd_sub_rem.append(psd_rem)

   # arrays of psds[[epochs,channels,freqs]] corresponding to each stage for all subjects
   psd_sub_w = np.array([np.array(psd_w) for psd_w in psd_sub_w])
   psd_sub_n1 = np.array([np.array(psd_n1) for psd_n1 in psd_sub_n1])
   psd_sub_n2 = np.array([np.array(psd_n2) for psd_n2 in psd_sub_n2])
   psd_sub_n3 = np.array([np.array(psd_n3) for psd_n3 in psd_sub_n3])
   psd_sub_rem = np.array([np.array(psd_rem) for psd_rem in psd_sub_rem])


5. Model Fitting: FOOOF 
~~~~~~~~~~~~~~~~~~~~~~~~~

The following blocks use the FOOOF Python library to model and extract the aperiodic and periodic components of the power spectral density (PSD). 

a. Wake sleepstage
+++++++++++++++++++

This block computes the FOOOF (Fitting Oscillations and One-Over F) model on the data corresponding to the "WAKE" sleep stage.

This block performs the following steps:
 
i. Initialize the FOOOF object(``fm``):

Initialize once for all sleepstages. Specify parameters for aperiodic mode, minimum peak height, and maximum number of peaks.

ii. Iterate through each PSG file and corresponding PSD data: 

Store filename labels (``subject_sub_w``) and FOOOF parameters(``aperiodic_params_sub_w``, ``periodic_params_sub_w``) for all subject files. Store additional labels for epoch and channel (``epoch_sub_w``, ``channel_sub_w``). These lists are initialized in the beginning of the block.
 
 Note: A single set of parameters is stored for each channel of individual epochs corresponding to PSD data for each subject file. Set of parameters to be stored: Aperiodic components- *Exponent, Offset, R^2, Error* and Periodic Components- *no of peaks picked, peak parameters*

iii. Fit the FOOOF model(``fm.fit()``) to the PSD data and retrieve the results(``res_w``) for each epoch and channel of a single subject's PSD data.

iv.  Store parameters:

- Store epoch (``epoch_w``), channel (``channel_w``) and filename labels (``subject_w``) for each epoch and channel.

- Separately store the aperiodic and periodic parameters for each epoch and channel from ``res_w``. 

- These will be iteratively stored in nested list structure to (``aperiodic_params_sub_w``, ``periodic_params_sub_w``)

 Note: We encountered epochs where peaks could not be picked. Periodic parameters are not retrieved from the model fit for such epochs. Guassian cannot be fitted for such epochs and the code throws an error. For such cases, n/a values should be assigned to periodic parameters to maintain uniformity of data.
 
- Additionally, maximum no of peaks(``max_n_peaks``) to be picked have been set to a limit of 10. However, no. of peaks picked from the model fit in each epoch may vary. For maintaining uniformity, the rest of the peak parameters are assigned with n/a values.

v. Organize the parameters into separate DataFrames, one for aperiodic parameters and the other for periodic parameters.

- Insert labels for each set of parameters to these dataframes

- Note that peak labels need to be added to periodic parameters, a single label for 3 adjacent columns(representing center frequency, peak power and bandwidth)

vi. Concatenate the DataFrames and compiles the data into a single DataFrame for the "WAKE" sleep stage. Insert an additional sleepstage label as well. 

 Note: The labels explicitely assigned to each set of parameters makes it easy to trace the parameters back to their origin.

vii. Save the final DataFrame to a CSV file for further analysis and interpretation.

.. code-block:: python
   
   #%% FOOOF | Block 4

   # by the end of this loop i should have -
   # complete data with 'none' values for non detectable guassians
   # separted periodic and aperiodic parameters compiled in dfs
   # a single dataframe for sleepstage parameters

   # initializing fooof
   fm = FOOOF(aperiodic_mode='fixed', min_peak_height=0.5, max_n_peaks=10)
   
   # WAKE SLEEPSTAGE

   # Initialize empty lists to store data
   epoch_sub_w = []
   channel_sub_w = []
   aperiodic_params_sub_w = []
   periodic_params_sub_w = []
   subject_sub_w = []

   # Outer loop
   for psd_w, file in zip(psd_sub_w, psg_files):

       # initializing empty lists for storing parameters
       epoch_w = []
       channel_w = []
       aperiodic_params_w = []
       periodic_params_w = []
       subject_w = []

       # Inner loop
       # looping over epochs and channel,outcome- epoch x channel no of data entries
       # for each subject psd
       for epoch in range (psd_w.shape[0]):
           for channel in range(psd_w.shape[1]):

               temp_periodic = [] # stores periodic params temporarily

               # fitting spectra
               fm.fit(freqs, psd_w[epoch, channel, :])

               # get results
               res_w = fm.get_results()

               # Labels
               # updating epoch x channel vals, filename
               epoch_w.append (f"w_Epoch_{epoch+1}")
               channel_w.append(f"Channel_{channel+1}")
               subject_w.append(file)

               # Aperiodic Component
               # append aperiodic vals to a list
               aperiodic_params_w.append([res_w.aperiodic_params[0],
                                 res_w.aperiodic_params[1],
                                 res_w.r_squared,
                                 res_w.error])

               # Periodic Component
               # editing out data for which peaks can't be picked
               if len(res_w.gaussian_params) == 0:

                   # n/a vals for periodic params in this case
                   for _ in range(0,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_w.append(temp_periodic)

               else:

                   # accessing the nested lists within peak_params list gives the no of peaks detected
                   no_of_peaks =np.shape(res_w.peak_params)[0]

                   # appending peak vals and n/a vals to fill for empty peak vals
                   for peak in range(no_of_peaks):
                       peak_pack = res_w.peak_params[peak]
                       for items in peak_pack:
                           temp_periodic.append(items)

                   # using throw away variale '_' for appending n/a vals, here max peaks=10
                   for _ in range(no_of_peaks,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_w.append(temp_periodic)

       # append master lists for each subject psd
       epoch_sub_w.append(epoch_w)
       channel_sub_w.append(channel_w)
       aperiodic_params_sub_w.append(aperiodic_params_w)
       periodic_params_sub_w.append(periodic_params_w)
       subject_sub_w.append(subject_w)

   # Dataframes

   # make df of periodic_params and add peak labels
   # Specify peak labels
   peak_labels = ['peak1', 'peak2', 'peak3', 'peak4', 'peak5',
     'peak6', 'peak7', 'peak8', 'peak9', 'peak10']

   for i in range(len(periodic_params_sub_w)):
       periodic_params_w = pd.DataFrame(periodic_params_sub_w[i])

   # Add peak labels
       for j in range(len(peak_labels)):
           peak_no = periodic_params_w.columns[j * 3: (j + 1) * 3]
           heading = peak_labels[j]
           periodic_params_w.rename(columns={col: heading for col in peak_no},
                               inplace=True)

       # Insert labels
       periodic_params_w.insert(0, 'Subject', pd.Series(subject_sub_w[i]))
       periodic_params_w.insert(1, 'Epoch', pd.Series(epoch_sub_w[i]))
       periodic_params_w.insert(2, 'Channel', pd.Series(channel_sub_w[i]))

       periodic_params_sub_w[i] = periodic_params_w

    # make a df of aperiodic_params and add parameter labels
    for i in range(len(aperiodic_params_sub_w)):
       aperiodic_params_w = pd.DataFrame(aperiodic_params_sub_w[i])
       aperiodic_params_w.columns = ['Exponent', 'Offset', 'R^2', 'Error']

       # Insert labels
       aperiodic_params_w.insert(0, 'Subject', pd.Series(subject_sub_w[i]))
       aperiodic_params_w.insert(1, 'Epoch', pd.Series(epoch_sub_w[i]))
       aperiodic_params_w.insert(2, 'Channel', pd.Series(channel_sub_w[i]))

       aperiodic_params_sub_w[i] = aperiodic_params_w

   # Concatenate the DataFrames within aperiodic_params_sub_w and periodic_params_sub_w
   aperiodic_params_sub_w = pd.concat(aperiodic_params_sub_w)
   periodic_params_sub_w = pd.concat(periodic_params_sub_w)

   # compiling data into a single wake dataframe
   report_w = pd.merge(aperiodic_params_sub_w, periodic_params_sub_w,
                  on=['Epoch', 'Channel', 'Subject'])
   #Insert sleepstage label
   report_w.insert(0, 'Stage', 'W')

   #saving to csv file
   report_w.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_wake.csv', index= False)

**The same code structure has been followed for N1, N2, N3 and REM sleepstage with their respective variables and labels**

b. N1 Sleepstage
+++++++++++++++++

.. code-block:: python

   #%% N1 STAGE | Block 5

   # Initialize empty lists to store data
   epoch_sub_n1 = []
   channel_sub_n1 = []
   aperiodic_params_sub_n1 = []
   periodic_params_sub_n1 = []
   subject_sub_n1 = []

   # Outer loop
   for psd_n1, file in zip(psd_sub_n1, psg_files):

       # initializing empty lists for storing parameters
       epoch_n1 = []
       channel_n1 = []
       aperiodic_params_n1 = []
       periodic_params_n1 = []
       subject_n1 = []

       # Inner loops
       # looping over epochs and channel for each subject psd
       for epoch in range (psd_n1.shape[0]):
           for channel in range(psd_n1.shape[1]):

               temp_periodic = [] # stores periodic params temporarily

               # fitting spectra
               fm.fit(freqs, psd_n1[epoch, channel, :])

               # get results
               res_n1 = fm.get_results()

               #Labels
               # updating epoch x channel vals, filename
               epoch_n1.append (f"n1_Epoch_{epoch+1}")
               channel_n1.append(f"Channel_{channel+1}")
               subject_n1.append(file)

               # Aperiodic Component
               # append aperiodic vals to a list
               aperiodic_params_n1.append([res_n1.aperiodic_params[0],
                                 res_n1.aperiodic_params[1],
                                 res_n1.r_squared,
                                 res_n1.error])

               # Periodic Component
               # editing out data for which peaks can't be picked
               if len(res_n1.gaussian_params) == 0:

                   # n/a vals for periodic params in this case
                   for _ in range(0,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_n1.append(temp_periodic)

               else:

                   # accessing the nested lists within peak_params list gives the no of peaks detected
                   no_of_peaks =np.shape(res_n1.peak_params)[0]

                   # appending peak vals and n/a vals to fill for empty peak vals
                   for peak in range(no_of_peaks):
                       peak_pack = res_n1.peak_params[peak]
                       for items in peak_pack:
                           temp_periodic.append(items)

                   # using throw away variale '_' for appending n/a vals, here max peaks=10
                   for _ in range(no_of_peaks,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_n1.append(temp_periodic)

       # append master lists for each subject psd
       epoch_sub_n1.append(epoch_n1)
       channel_sub_n1.append(channel_n1)
       aperiodic_params_sub_n1.append(aperiodic_params_n1)
       periodic_params_sub_n1.append(periodic_params_n1)
       subject_sub_n1.append(subject_n1)

   # Dataframes

   # make df of periodic_params and add peak labels
   for i in range(len(periodic_params_sub_n1)):
       periodic_params_n1 = pd.DataFrame(periodic_params_sub_n1[i])

   # Add peak labels
       for j in range(len(peak_labels)):
           peak_no = periodic_params_n1.columns[j * 3: (j + 1) * 3]
           heading = peak_labels[j]
           periodic_params_n1.rename(columns={col: heading for col in peak_no},
                               inplace=True)
       #Insert labels
       periodic_params_n1.insert(0, 'Subject', pd.Series(subject_sub_n1[i]))
       periodic_params_n1.insert(1, 'Epoch', pd.Series(epoch_sub_n1[i]))
       periodic_params_n1.insert(2, 'Channel', pd.Series(channel_sub_n1[i]))

       periodic_params_sub_n1[i] = periodic_params_n1

   # make a df of aperiodic_params and add parameter labels
   for i in range(len(aperiodic_params_sub_n1)):
       aperiodic_params_n1 = pd.DataFrame(aperiodic_params_sub_n1[i])
       aperiodic_params_n1.columns = ['Exponent', 'Offset', 'R^2', 'Error']

       #Insert labels
       aperiodic_params_n1.insert(0, 'Subject', pd.Series(subject_sub_n1[i]))
       aperiodic_params_n1.insert(1, 'Epoch', pd.Series(epoch_sub_n1[i]))
       aperiodic_params_n1.insert(2, 'Channel', pd.Series(channel_sub_n1[i]))

       aperiodic_params_sub_n1[i] = aperiodic_params_n1

   # Concatenate the DataFrames within aperiodic_params_sub_w and periodic_params_sub_w
   aperiodic_params_sub_n1 = pd.concat(aperiodic_params_sub_n1)
   periodic_params_sub_n1 = pd.concat(periodic_params_sub_n1)

   # compiling data into a single wake dataframe
   report_n1 = pd.merge(aperiodic_params_sub_n1, periodic_params_sub_n1,
                   on=['Epoch', 'Channel', 'Subject'])
   #Insert sleepstage label
   report_n1.insert(0, 'Stage', 'N1')

   #saving to csv file
   report_n1.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_n1.csv', index= False)

c. N2 Sleepstage
+++++++++++++++++ 

.. code-block:: python

   #%% N2 STAGE | Block 6

   # Initialize empty lists to store data
   epoch_sub_n2 = []
   channel_sub_n2 = []
   aperiodic_params_sub_n2 = []
   periodic_params_sub_n2 = []
   subject_sub_n2 = []

   # Outer loop
   for psd_n2, file in zip(psd_sub_n2, psg_files):

       # initializing empty lists for storing parameters
       epoch_n2 = []
       channel_n2 = []
       aperiodic_params_n2 = []
       periodic_params_n2 = []
       subject_n2 = []

       # Inner loops
       # looping over epochs and channel for each subject psd
       for epoch in range (psd_n2.shape[0]):
           for channel in range(psd_n2.shape[1]):

               temp_periodic = [] # stores periodic params temporarily

               # fitting spectra
               fm.fit(freqs, psd_n2[epoch, channel, :])

               # get results
               res_n2 = fm.get_results()

               # Labels
               # updating epoch x channel vals, filename
               epoch_n2.append (f"n2_Epoch_{epoch+1}")
               channel_n2.append(f"Channel_{channel+1}")
               subject_n2.append(file)

               # Aperiodic Component
               # append aperiodic vals to a list
               aperiodic_params_n2.append([res_n2.aperiodic_params[0], 
					res_n2.aperiodic_params[1],
					res_n2.r_squared,
					res_n2.error])

               # Periodic Component
               # editing out data for which peaks can't be picked
               if len(res_n2.gaussian_params) == 0:

                   # n/a vals for periodic params in this case
                   for _ in range(0,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_n2.append(temp_periodic)

               else:

                   # accessing the nested lists within peak_params list gives the no of peaks detected
                   no_of_peaks =np.shape(res_n2.peak_params)[0]

                   # appending peak vals and n/a vals to fill for empty peak vals
                   for peak in range(no_of_peaks):
                       peak_pack = res_n2.peak_params[peak]
                       for items in peak_pack:
                           temp_periodic.append(items)

                   # using throw away variale '_' for appending n/a vals, here max peaks=10
                   for _ in range(no_of_peaks,10):
                       for _ in range(3):
                           temp_periodic.append(np.nan)

                   periodic_params_n2.append(temp_periodic)

      # append master lists for each subject psd
       epoch_sub_n2.append(epoch_n2)
       channel_sub_n2.append(channel_n2)
       aperiodic_params_sub_n2.append(aperiodic_params_n2)
       periodic_params_sub_n2.append(periodic_params_n2)
       subject_sub_n2.append(subject_n2)

   # Dataframes

   # make df of periodic_params and add peak labels
   for i in range(len(periodic_params_sub_n2)):
       periodic_params_n2 = pd.DataFrame(periodic_params_sub_n2[i])

   # Add peak labels
       for j in range(len(peak_labels)):
           peak_no = periodic_params_n2.columns[j * 3: (j + 1) * 3]
           heading = peak_labels[j]
           periodic_params_n2.rename(columns={col: heading for col in peak_no},
                               inplace=True)

       # Insert labels
       periodic_params_n2.insert(0, 'Subject', pd.Series(subject_sub_n2[i]))
       periodic_params_n2.insert(1, 'Epoch', pd.Series(epoch_sub_n2[i]))
       periodic_params_n2.insert(2, 'Channel', pd.Series(channel_sub_n2[i]))

       periodic_params_sub_n2[i] = periodic_params_n2


   # make a df of aperiodic_params and add parameter labels
   for i in range(len(aperiodic_params_sub_n2)):
       aperiodic_params_n2 = pd.DataFrame(aperiodic_params_sub_n2[i])
       aperiodic_params_n2.columns = ['Exponent', 'Offset', 'R^2', 'Error']

       # Insert labels
       aperiodic_params_n2.insert(0, 'Subject', pd.Series(subject_sub_n2[i]))
       aperiodic_params_n2.insert(1, 'Epoch', pd.Series(epoch_sub_n2[i]))
       aperiodic_params_n2.insert(2, 'Channel', pd.Series(channel_sub_n2[i]))

       aperiodic_params_sub_n2[i] = aperiodic_params_n2

   # Concatenate the DataFrames within aperiodic_params_sub_w and periodic_params_sub_w
   aperiodic_params_sub_n2 = pd.concat(aperiodic_params_sub_n2)
   periodic_params_sub_n2 = pd.concat(periodic_params_sub_n2)

   # compiling data into a single wake dataframe
   report_n2 = pd.merge(aperiodic_params_sub_n2, periodic_params_sub_n2,
                   on=['Epoch', 'Channel', 'Subject'])
   # Insert slepstage label
   report_n2.insert(0, 'Stage', 'N2')

   #saving to csv file
   report_n2.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_n2.csv', index= False)


d. N3 Sleepstage
+++++++++++++++++

.. code-block:: python

	#%% N3 STAGE | Block 7

	# Initialize empty lists to store data
	epoch_sub_n3 = []
	channel_sub_n3 = []
	aperiodic_params_sub_n3 = []
	periodic_params_sub_n3 = []
	subject_sub_n3 = []

	# Outer loop
	for psd_n3, file in zip(psd_sub_n3, psg_files):

		 # initializing empty lists for storing parameters
		 epoch_n3 = []
		 channel_n3 = []
		 aperiodic_params_n3 = []
		 periodic_params_n3 = []
		 subject_n3 = []

		 # Inner loops
		 # looping over epochs and channel for each subject psd
		 for epoch in range (psd_n3.shape[0]):
			  for channel in range(psd_n3.shape[1]):

					temp_periodic = [] # stores periodic params temporarily

					# fitting spectra
					fm.fit(freqs, psd_n3[epoch, channel, :])

					# get results
					res_n3 = fm.get_results()

					# Labels
					# updating epoch x channel vals, filename
					epoch_n3.append (f"n3_Epoch_{epoch+1}")
					channel_n3.append(f"Channel_{channel+1}")
					subject_n3.append(file)

					# Aperiodic Component
					# append aperiodic vals to a list
					aperiodic_params_n3.append([res_n3.aperiodic_params[0],
									res_n3.aperiodic_params[1],
									res_n3.r_squared,
									res_n3.error])

					# Periodic Component
					# editing out data for which peaks can't be picked
					if len(res_n3.gaussian_params) == 0:

						 # n/a vals for periodic params in this case
						 for _ in range(0,10):
							  for _ in range(3):
									temp_periodic.append(np.nan)

						 periodic_params_n3.append(temp_periodic)

					else:

						 # accessing the nested lists within peak_params list gives the no of peaks detected
						 no_of_peaks =np.shape(res_n3.peak_params)[0]

						 # appending peak vals and n/a vals to fill for empty peak vals
						 for peak in range(no_of_peaks):
							  peak_pack = res_n3.peak_params[peak]
							  for items in peak_pack:
									temp_periodic.append(items)

						 # using throw away variale '_' for appending n/a vals, here max peaks=10
						 for _ in range(no_of_peaks,10):
							  for _ in range(3):
									temp_periodic.append(np.nan)

						 periodic_params_n3.append(temp_periodic)

		 # append master lists for each subject psd
		 epoch_sub_n3.append(epoch_n3)
		 channel_sub_n3.append(channel_n3)
		 aperiodic_params_sub_n3.append(aperiodic_params_n3)
		 periodic_params_sub_n3.append(periodic_params_n3)
		 subject_sub_n3.append(subject_n3)

	# Dataframes

	# make df of periodic_params and add peak labels
	for i in range(len(periodic_params_sub_n3)):

		 periodic_params_n3 = pd.DataFrame(periodic_params_sub_n3[i])

	# Add peak labels
		 for j in range(len(peak_labels)):
			  peak_no = periodic_params_n3.columns[j * 3: (j + 1) * 3]
			  heading = peak_labels[j]
			  periodic_params_n3.rename(columns={col: heading for col in peak_no},
			                            inplace=True)

		 # Insert labels
		 periodic_params_n3.insert(0, 'Subject', pd.Series(subject_sub_n3[i]))
		 periodic_params_n3.insert(1, 'Epoch', pd.Series(epoch_sub_n3[i]))
		 periodic_params_n3.insert(2, 'Channel', pd.Series(channel_sub_n3[i]))

		 periodic_params_sub_n3[i] = periodic_params_n3

	# make a df of aperiodic_params and add parameter labels
	for i in range(len(aperiodic_params_sub_n3)):

		 aperiodic_params_n3 = pd.DataFrame(aperiodic_params_sub_n3[i])
		 aperiodic_params_n3.columns = ['Exponent', 'Offset', 'R^2', 'Error']

		 # Insert labels
		 aperiodic_params_n3.insert(0, 'Subject', pd.Series(subject_sub_n3[i]))
		 aperiodic_params_n3.insert(1, 'Epoch', pd.Series(epoch_sub_n3[i]))
		 aperiodic_params_n3.insert(2, 'Channel', pd.Series(channel_sub_n3[i]))

		 aperiodic_params_sub_n3[i] = aperiodic_params_n3

	# Concatenate the DataFrames within aperiodic_params_sub_w and periodic_params_sub_w
	aperiodic_params_sub_n3 = pd.concat(aperiodic_params_sub_n3)
	periodic_params_sub_n3 = pd.concat(periodic_params_sub_n3)

	# compiling data into a single wake dataframe
	report_n3 = pd.merge(aperiodic_params_sub_n3, periodic_params_sub_n3,
	            on=['Epoch', 'Channel', 'Subject'])
	# Insert sleepstage labels
	report_n3.insert(0, 'Stage', 'N3')

	#saving to csv file
	report_n3.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_n3.csv', index= False)

e. REM Sleepstage
+++++++++++++++++

.. code-block:: python

	#%% REM STAGE | Block 8

	# Initialize empty lists to store data
	epoch_sub_rem = []
	channel_sub_rem = []
	aperiodic_params_sub_rem = []
	periodic_params_sub_rem = []
	subject_sub_rem = []

	# Outer loop
	for psd_rem, file in zip(psd_sub_rem, psg_files):

		 # initializing empty lists for storing parameters
		 epoch_rem = []
		 channel_rem = []
		 aperiodic_params_rem = []
		 periodic_params_rem = []
		 subject_rem = []

		 # Inner loops
		 # looping over epochs and channel,outcome- epoch x channel no of data entries 
		 # for each subject psd
		 for epoch in range (psd_rem.shape[0]):
			  for channel in range(psd_rem.shape[1]):

					temp_periodic = [] # stores periodic params temporarily

					# fitting spectra
					fm.fit(freqs, psd_rem[epoch, channel, :])

					# get results
					res_rem = fm.get_results()

					# Labels
					# updating epoch x channel vals, filename
					epoch_rem.append (f"rem_Epoch_{epoch+1}")
					channel_rem.append(f"Channel_{channel+1}")
					subject_rem.append(file)

					# Aperiodic Component
					# append aperiodic vals to a list
					aperiodic_params_rem.append([res_rem.aperiodic_params[0],
									res_rem.aperiodic_params[1],
									res_rem.r_squared,
									res_rem.error])

					# Periodic Component
					# editing out data for which peaks can't be picked
					if len(res_rem.gaussian_params) == 0:

						 # n/a vals for periodic params in this case
						 for _ in range(0,10):
							  for _ in range(3):
									temp_periodic.append(np.nan)

						 periodic_params_rem.append(temp_periodic)

					else:

						 # accessing the nested lists within peak_params list gives the no of peaks detected
						 no_of_peaks =np.shape(res_rem.peak_params)[0]

						 # appending peak vals and n/a vals to fill for empty peak vals
						 for peak in range(no_of_peaks):
							  peak_pack = res_rem.peak_params[peak]
							  for items in peak_pack:
									temp_periodic.append(items)

						 # using throw away variale '_' for appending n/a vals, here max peaks=10
						 for _ in range(no_of_peaks,10):
							  for _ in range(3):
									temp_periodic.append(np.nan)

						 periodic_params_rem.append(temp_periodic)

		# append master lists for each subject psd
		 epoch_sub_rem.append(epoch_rem)
		 channel_sub_rem.append(channel_rem)
		 aperiodic_params_sub_rem.append(aperiodic_params_rem)
		 periodic_params_sub_rem.append(periodic_params_rem)
		 subject_sub_rem.append(subject_rem)

	# Dataframes

	# make df of periodic_params and add peak labels
	for i in range(len(periodic_params_sub_rem)):
		 periodic_params_rem = pd.DataFrame(periodic_params_sub_rem[i])

	# Add peak labels
		 for j in range(len(peak_labels)):
			  peak_no = periodic_params_rem.columns[j * 3: (j + 1) * 3]
			  heading = peak_labels[j]
			  periodic_params_rem.rename(columns={col: heading for col in peak_no},
			                             inplace=True)

		 # Insert labels
		 periodic_params_rem.insert(0, 'Subject', pd.Series(subject_sub_rem[i]))
		 periodic_params_rem.insert(1, 'Epoch', pd.Series(epoch_sub_rem[i]))
		 periodic_params_rem.insert(2, 'Channel', pd.Series(channel_sub_rem[i]))

		 periodic_params_sub_rem[i] = periodic_params_rem

	# make a df of aperiodic_params and add parameter labels
	for i in range(len(aperiodic_params_sub_rem)):
		 aperiodic_params_rem = pd.DataFrame(aperiodic_params_sub_rem[i])
		 aperiodic_params_rem.columns = ['Exponent', 'Offset', 'R^2', 'Error']

		 # Insert labels
		 aperiodic_params_rem.insert(0, 'Subject', pd.Series(subject_sub_rem[i]))
		 aperiodic_params_rem.insert(1, 'Epoch', pd.Series(epoch_sub_rem[i]))
		 aperiodic_params_rem.insert(2, 'Channel', pd.Series(channel_sub_rem[i]))

		 aperiodic_params_sub_rem[i] = aperiodic_params_rem

	# Concatenate the DataFrames within aperiodic_params_sub_w and periodic_params_sub_w
	aperiodic_params_sub_rem = pd.concat(aperiodic_params_sub_rem)
	periodic_params_sub_rem = pd.concat(periodic_params_sub_rem)

	# compiling data into a single wake dataframe
	report_rem = pd.merge(aperiodic_params_sub_rem, periodic_params_sub_rem,
	             on=['Epoch', 'Channel', 'Subject'])
	# Insert sleepstage labels
	report_rem.insert(0, 'Stage', 'REM')

	#saving to csv file
	report_rem.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_rem.csv', index= False)

6. Compiling Data
~~~~~~~~~~~~~~~~~~~

This block of code compiles all sleepstage dataframes into a single dataframe ``report_sleepstages``.
Remove data with ``R^2`` value below 0.9. The block makes a dataframe solely for the Aperiodic Component of EEG Data which will be used in analysis. 

.. code-block:: python

	#%% Compile all sleepstages into one | Block 9

	#compile dataframe and save it to a csv file
	report_sleepstages = pd.concat([report_w,report_n1,report_n2,report_n3,report_rem],
									axis=0)
	report_sleepstages.reset_index(drop=True, inplace= True)

	#saving to csv file
	report_sleepstages.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_fooof_sleepdata.csv', index= False)

	#remove periodic params and entries with r_squred vals <0.9
	report_sleepstages_II = report_sleepstages.drop(report_sleepstages.columns[8:38], axis=1)
	report_sleepstages_II = report_sleepstages_II[report_sleepstages_II['R^2'] >= 0.9]

	report_sleepstages_II.reset_index(drop=True, inplace= True)
	
7. Averaging across Epochs 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This block computes the average values and trimmed mean across epochs for each sleep stage and channel.

* Group by stage and channel, then take the mean of the aperiodic and periodic components for all epochs.

* Compute the trimmed mean for each column (``Exponent``, ``Offset``, ``R^2``, ``Error``) in the ``report_sleepstages_II`` DataFrame, and add columns containing the trimmed mean values to the ``Channel_avg_vals`` DataFrame.

* Reset the index of the ``Channel_avg_vals`` DataFrame to convert the grouped columns ('Channel' and 'Stage') back to regular columns.

* Extract the channel number from the ``Channel`` column, convert it to an integer and update the ``Channel`` column in ``Channel_avg_vals``.

* Sort the DataFrame by ``Channel`` and ``Stage`` in ascending order.

* Again reset the index of ``Channel_avg_vals`` and drop the old index to get a clean DataFrame containing the averaged values and trimmed means for each channel and sleep stage.

.. code-block:: python

	#%% AVERAGING ACROSS EPOCHS | Block 10

	# Trimmed mean
	# Define the trim percentage (here, 10%)
	trim_percentage = 0.1

	#  Group by columns and calculate trimmed mean for each group
	Channel_avg_vals = report_sleepstages_II.groupby(['Stage', 'Channel']).apply(lambda group: group.iloc[:, 2:].apply(trim_mean, proportiontocut=trim_percentage))
	
	#reset the index to convert the grouped columns ('Channel' and 'Stage') back to regular columns
	Channel_avg_vals = Channel_avg_vals.reset_index()

	# Extracting the channel number from the 'Channel' column
	Channel_avg_vals['Channel'] = Channel_avg_vals['Channel'].str.split('_').str[1].astype(int)

	# Sorting the DataFrame by 'Channel' and 'Stage'
	Channel_avg_vals = Channel_avg_vals.sort_values(by=[ 'Stage','Channel'], ascending=[True, True])
	Channel_avg_vals.reset_index(drop= True, inplace= True)

8. Plotting Results
~~~~~~~~~~~~~~~~~~~~

a. Topoplot
+++++++++++++

The following blocks generates topoplots for the exponent and offset values corresponding to 19 selected channels and 5 sleep stages. 

The topoplots provide a visual representation of the distribution of these values across different channels and sleep stages.

**Parameters for Topoplot (Block 11):**

Specify the channels to be included in the topoplot using the channels_to_pick_topo list.

The code creates 2D dataframes (``Exponent_vals`` and ``Offset_vals``) containing the exponent and offset values for each channel across the five sleep stages. 

The index of the dataframes is set to the channel names for visualization using ``yasa``.

**Topoplot Generation (Block 12):**

* Exponent Topoplots

The code iterates through the sleep stages and generates topoplots for the exponent values.

For each stage, we define the color scale using the maximum and minimum exponent values across all channels and stages.

The ``yasa.topoplot`` function is then called to create the topoplot, and ``plt.show()`` displays it.

Additionally, the topoplot is saved as an image file in the specified location.

* Offset Topoplots

Similarly, the code generates topoplots for the offset values. The process is similar to that of the exponent topoplots, with appropriate color scaling, plotting, and saving.

.. code-block:: python

	#%% Parameters for Topoplot | Block 11

	channels_to_pick_topo = ['Fp1', 'Fp2', 'F3', 'F4', 'C3', 'C4', 'P3', 'P4', 'O1', 'O2',
	'F7', 'F8', 'T3', 'T4', 'Fz', 'Cz', 'Pz', 'A1', 'A2']

	# Exponet vals

	# make a 2D dataframe containing exponent vals corresponding to 19 channels for 5 sleepstages
	Exponent_vals = Channel_avg_vals.pivot(index='Channel', columns='Stage', values='Exponent')

	# The index MUST be the channel names for yasa
	Exponent_vals.index = channels_to_pick_topo

	# Offset vals

	# make a 2D dataframe containing offset vals corresponding to 19 channels for 5 sleepstages
	Offset_vals = Channel_avg_vals.pivot(index='Channel', columns='Stage', values='Offset')

	# The index MUST be the channel names for yasa
	Offset_vals.index = channels_to_pick_topo

.. code-block:: python

	#%% TOPOPLOT | Block 12

	# define sleep_stages
	sleep_stages = ['W','N1','N2','N3','REM']

	# EXPONENT TOPO

	# loop over sleep stages and plot the data# Create a 3-D array
	for i in range(0,len(sleep_stages)):
		vmax = Exponent_vals.max().max()
		vmin = Exponent_vals.min().min()
		stage = sleep_stages[i]
		yasa.topoplot(Exponent_vals[stage], title =stage,
                  vmin= vmin,
                  vmax= vmax,
                  cmap = 'coolwarm',
                  n_colors= 10 )
		plt.tight_layout() #adjusts layout of plot
		plt.show()
		plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
                + 'Exponent_allsubs_' + stage , facecolor='white')
		plt.close()

	# OFFSET TOPO

	# loop over sleep stages and plot the data
	for i in range(0,len(sleep_stages)):
		vmax = Offset_vals.max().max()
		vmin = Offset_vals.min().min()
		stage = sleep_stages[i]
		yasa.topoplot(Offset_vals[stage], title =stage,
                  vmin= vmin,
                  vmax= vmax,
                  cmap = 'coolwarm',
                  n_colors= 10 )
		plt.tight_layout() #adjusts layout of plot
		plt.show()
		plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
                + 'Offset_allsubs_' + stage , facecolor='white')
		plt.close()


b. Scatter Plotting
++++++++++++++++++++

The following code blocks demonstrate the visualization of the aperiodic parameters using scatter plots. These plots represent the relationship between the ``Offset`` and ``Exponent`` values across sleepstages and channels.

**Scatter Plot (Block 13)**

The code creates a scatter plot by plotting ``Offset`` values on the x-axis and ``Exponent`` values on the y-axis. The plot represents the general distribution of the aperiodic parameters. The x and y axes are labeled appropriately, and the plot is given a title. Finally, the plot is displayed and saved as an image file.

**Scatter Plot with Regression Line (Block 14)**

In addition to the scatter plot, the code creates a scatter plot with a regression line using the Seaborn library. The regression line represents the overall trend in the relationship between ``Offset`` and ``Exponent`` values. The plot is displayed and saved as an image file for further reference.

.. code-block:: python

	#%%  Finally, we can plot aperiodic parameters | Block 13

	Exp = Channel_avg_vals['Exponent']
	Offs = Channel_avg_vals['Offset']

	# Scatter plot
	plt.scatter(Offs, Exp)
	plt.xlabel('Offset')
	plt.ylabel('Exponent')
	plt.title('Aperiodic Parameters')
	plt.show()
	plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' + 'Aperiodic_params',
            facecolor='white')

.. code-block:: python

	#%% Scatter plot with regression line | Block 14

	sb.regplot(data = Channel_avg_vals, x= Offs, y= Exp, color= 'blue')

	plt.show()
	plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' + 'Aperiodic_params_reg',
            facecolor='white')




