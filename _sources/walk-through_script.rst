
Walk-through script
--------------------

Here is a script which will walk you through the process of computing aperiodic component of Sleep EEG data. It follows the same approach as the main code script. Any details about the codes used can be referred to the **Code Script** section. This script computes for a single PSG file.

.. code-block:: python

	#%% Load the libraries
	from tkinter.filedialog import askopenfilenames
	from tkinter import filedialog
	from tkinter import Tk
	import yasa
	import numpy as np
	import matplotlib.pyplot as plt
	from scipy.signal import welch
	from scipy.stats import trim_mean
	import mne
	import pandas as pd
	from fooof import FOOOF
	Tk().withdraw() # we don't want a full GUI, so keep the root window from appearing
	#%% Setup the folders
	PSGfiles = askopenfilenames(title = "Select PSG data files",
	                            filetypes = (("EDF file","*.edf"),
	                            ('All files', '*.*')))

	scored_files = askopenfilenames(title = "Select Scored files",
	                                filetypes = (("EDF file","*.edf"),
	                                ('All files', '*.*')))

	#%% Cleanup the hypnogram data into a sequence of stages every epoch

	hypnogram = mne.read_annotations(scored_files[0])
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
	hypno_30s = []
	for stages in range(len(hypnogram_annot)):
		 for repetitions in range(int(hypnogram_annot.duration[stages])):
			  hypno_30s.append(hypnogram_annot.description[stages])

	#%% Load the data | Get the 19 channels required + A1 and A2
	edfdata = mne.io.read_raw_edf(PSGfiles[0], preload=True)
	srate = int(edfdata.info['sfreq'])

	channels_to_pick = ['Fp1', 'Fp2', 'F3', 'F4', 'C3', 'C4', 'P3', 'P4', 'O1', 'O2',
	 'F7', 'F8', 'T3', 'T4', 'Fz', 'Cz', 'Pz', 'A1', 'A2']

	edfdata.pick_channels(channels_to_pick)

	# bandpass filter data
	edfdata.filter(0.1,None,fir_design='firwin').load_data()
	edfdata.filter(None,45,fir_design='firwin').load_data()

	data = edfdata.get_data() * 1e6 #coverting volts to microvolts

	#%% Generate 30 seconds PSDs for all channels
	# Create a 3-D array

	# cut data into 30 seconds epochs
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

	#%% psds correspnding to respective sleepstages
	psd_w = []
	psd_n1 = []
	psd_n2 = []
	psd_n3 = []
	psd_rem = []

	# index by index matching of elements in PSD and hypnos_30
	for i, sleepstage in zip(psd,hypno_30s):
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

	# making an array of psd[epochs,channels,freqs]       
	psd_n1 = np.array(psd_n1)
	psd_n2 = np.array(psd_n2)
	psd_n3 = np.array(psd_n3)
	psd_rem = np.array(psd_rem)
	psd_w = np.array(psd_w)       

	#%% In the next code blocks, we are going to 
	# run fooof for each channel for every epoch corresponding to a single sleepstage.
	# The plots for fitted spectra will be saved in 'Results' file.
	# The fooof parameters would be stored in a dataframe for each sleepstage.
	# By the end of it we will have fooof parameter data for four sleep stages.
	# Compile these four dataframes into one sleep dataframe with relevant labels 
	# Labels- subject,sleepstage, epoch_channel, foooof parameters
	# Sort the dataframe by r_squared vals and delete entries with r_squared vals < 0.9
	# Finally you should have a big dataframe with numerous enteries for periodic and aperiodic parameters

	#%% FOOOF

	# by the end of this loop i should have - 
	# complete data with 'none' values for non detectable guassians
	# graphs of fitted power spectra (except for not detectale gaussians)
	# separted periodic and aperiodic parameters compiled in dfs
	# a single dataframe for wake sleepstage parameters

	# initializing fooof
	fm = FOOOF(aperiodic_mode='fixed', min_peak_height=0.5, max_n_peaks=10)

	# WAKE SLEEPSTAGE

	# initializing empty lists for storing parameters
	epoch_w=[] 
	channel_w=[]
	aperiodic_params_w=[]
	periodic_params_w=[]

	# looping over epochs and channel,outcome- epoch x channel no of data entries, here 190
	for epoch in range (40,50):
		 for channel in range(psd_w.shape[1]):

			  temp_periodic = [] # stores periodic params temporarily

			  # fitting spectra
			  fm.fit(freqs, psd_w[epoch, channel, :])

			  # get results
			  res_w = fm.get_results()

			  # append aperiodic vals to a list
			  aperiodic_params_w.append([res_w.aperiodic_params[0], 
				                         res_w.aperiodic_params[1],
				                         res_w.r_squared,
				                         res_w.error])

			  # updating epoch x channel vals
			  epoch_w.append (f"w_Epoch_{epoch+1}")
			  channel_w.append(f"Channel_{channel+1}")

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

					# plot the fitted spectra, assign results to fm 
					fm.results=res_w  
					fm.plot(plot_aperiodic=True, plot_peaks='shade', 
					        peak_kwargs={'color' : 'green'}, plt_log=False)
					plt.title(f"w, Epoch {epoch+1}, channel {channel+1}")
					plt.show()
					plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' 
					            + 'w_' + 'Epoch_' + str(epoch) + '_' 
					            + channels_to_pick[channel],
					            dpi = 600)
					plt.close()

	# make df of periodic_params and add peak labels
	periodic_params_w = pd.DataFrame(periodic_params_w)

	peak_labels =['peak1','peak2','peak3','peak4','peak5',
		 'peak6','peak7','peak8','peak9','peak10']

	for i in range(len(peak_labels)):
		 peak_no = periodic_params_w.columns[i * 3: (i + 1) * 3]
		 heading = peak_labels[i]
		 periodic_params_w.rename(columns={col: heading for col in peak_no},
										inplace=True)

	# make a df of aperiodic_params and add parameter labels
	aperiodic_params_w = pd.DataFrame(aperiodic_params_w)

	aperiodic_params_w.columns=['Exponent','Offset','R^2','Error'] 

	# compiling data into a single wake dataframe
	report_w=pd.merge(aperiodic_params_w, periodic_params_w,  left_index= True, right_index= True)

	# add epoch_channel labels
	report_w.insert(loc=0, column='Stage', value='W')
	report_w.insert(1, 'Channel',channel_w)
	report_w.insert(2, 'Epoch',epoch_w)

	#%% N1 SLEEPSTAGE

	# initializing empty lists for storing  parameters 
	epoch_n1=[] 
	channel_n1=[]
	aperiodic_params_n1=[]
	periodic_params_n1=[]

	# looping over epochs and channel,outcome- epoch x channel no of data entries, here 190
	for epoch in range (40,50):
		 for channel in range(psd_n1.shape[1]):

			  temp_periodic = [] # stores periodic params temporarily

			  # fitting spectra
			  fm.fit(freqs, psd_n1[epoch, channel, :])

			  # get results
			  res_n1 = fm.get_results()

			  # append aperiodic vals to a list
			  aperiodic_params_n1.append([res_n1.aperiodic_params[0], 
				                          res_n1.aperiodic_params[1],
				                          res_n1.r_squared,
				                          res_n1.error])

			  # updating epoch x channel vals
			  epoch_n1.append (f"n1_Epoch_{epoch+1}")
			  channel_n1.append(f"Channel_{channel+1}")

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

					# plot the fitted spectra, assign results to fm 
					fm.results=res_n1 
					fm.plot(plot_aperiodic=True, plot_peaks='shade', 
					        peak_kwargs={'color' : 'green'}, plt_log=False)
					plt.title(f"n1, Epoch {epoch+1}, channel {channel+1}")
					plt.show()
					plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' 
					            + 'n1_' + 'Epoch_' + str(epoch) + '_' 
					            + channels_to_pick[channel],
					            dpi = 600)
					plt.close()

	# make df of periodic_params and add peak labels
	periodic_params_n1 = pd.DataFrame(periodic_params_w)

	for i in range(len(peak_labels)):
		 peak_no = periodic_params_n1.columns[i * 3: (i + 1) * 3]
		 heading = peak_labels[i]
		 periodic_params_n1.rename(columns={col: heading for col in peak_no},
										inplace=True)

	# make a df of aperiodic_params and add parameter labels
	aperiodic_params_n1 = pd.DataFrame(aperiodic_params_n1)

	aperiodic_params_n1.columns=['Exponent','Offset','R^2','Error'] 

	# compiling data into a single wake dataframe
	report_n1=pd.merge(aperiodic_params_n1, periodic_params_n1,  left_index= True, right_index= True)

	# add epoch_channel labels
	report_n1.insert(loc=0, column='Stage', value='N1')
	report_n1.insert(1,'Channel',channel_n1)
	report_n1.insert(2, 'Epoch',epoch_n1)

	#%% N2 SLEEPSTAGE

	# initializing empty lists for storing parameters 
	epoch_n2=[] 
	channel_n2=[]
	aperiodic_params_n2=[]
	periodic_params_n2=[]

	# looping over epochs and channel,outcome- epoch x channel no of data entries, here 190
	for epoch in range (50,60):
		 for channel in range(psd_n2.shape[1]):

			  temp_periodic = [] # stores periodic params temporarily

			  # fitting spectra
			  fm.fit(freqs, psd_n2[epoch, channel, :])

			  # get results
			  res_n2 = fm.get_results()

			  # append aperiodic vals to a list
			  aperiodic_params_n2.append([res_n2.aperiodic_params[0], 
				                           res_n2.aperiodic_params[1],
				                           res_n2.r_squared,
				                           res_n2.error])

			  # updating epoch x channel vals
			  epoch_n2.append (f"n2_Epoch_{epoch+1}")
			  channel_n2.append(f"Channel_{channel+1}")

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

					# plot the fitted spectra, assign results to fm 
					fm.results=res_n2 
					fm.plot(plot_aperiodic=True, plot_peaks='shade', 
					        peak_kwargs={'color' : 'green'}, plt_log=False)
					plt.title(f"n2, Epoch {epoch+1}, channel {channel+1}")
					plt.show()
					plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' 
					            + 'n2_' + 'Epoch_' + str(epoch) + '_' 
					            + channels_to_pick[channel],
					            dpi = 600)
					plt.close()

	# make df of periodic_params and add peak labels
	periodic_params_n2 = pd.DataFrame(periodic_params_n2)

	for i in range(len(peak_labels)):
		 peak_no = periodic_params_n2.columns[i * 3: (i + 1) * 3]
		 heading = peak_labels[i]
		 periodic_params_n2.rename(columns={col: heading for col in peak_no},
										inplace=True)

	# make a df of aperiodic_params and add parameter labels
	aperiodic_params_n2 = pd.DataFrame(aperiodic_params_n2)

	aperiodic_params_n2.columns=['Exponent','Offset','R^2','Error'] 

	# compiling data into a single wake dataframe
	report_n2=pd.merge(aperiodic_params_n2, periodic_params_n2,  left_index= True, right_index= True)

	# add epoch_channel labels
	report_n2.insert(loc=0, column='Stage', value='N2')
	report_n2.insert(1, 'Channel',channel_n2)
	report_n2.insert(2, 'Epoch',epoch_n2)

	#%% N3 SLEEPSTAGE

	# initializing empty lists for storing  parameters 
	epoch_n3=[] 
	channel_n3=[]
	aperiodic_params_n3=[]
	periodic_params_n3=[]

	# looping over epochs and channel,outcome- epoch x channel no of data entries, here 190
	for epoch in range (20,30):
		 for channel in range(psd_n3.shape[1]):

			  temp_periodic = [] # stores periodic params temporarily

			  # fitting spectra
			  fm.fit(freqs, psd_n3[epoch, channel, :])

			  # get results
			  res_n3 = fm.get_results()

			  # append aperiodic vals to a list
			  aperiodic_params_n3.append([res_n3.aperiodic_params[0], 
				                          res_n3.aperiodic_params[1],
				                          res_n3.r_squared,
				                          res_n3.error])

			  # updating epoch x channel vals
			  epoch_n3.append (f"n3_Epoch_{epoch+1}")
			  channel_n3.append(f"Channel_{channel+1}")

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

					# plot the fitted spectra, assign results to fm 
					fm.results=res_n3  
					fm.plot(plot_aperiodic=True, plot_peaks='shade', 
					        peak_kwargs={'color' : 'green'}, plt_log=False)
					plt.title(f"n3, Epoch {epoch+1}, channel {channel+1}")
					plt.show()
					plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' 
					            + 'n3_' + 'Epoch_' + str(epoch) + '_' 
					            + channels_to_pick[channel],
					            dpi = 600)
					plt.close()

	# make df of periodic_params and add peak labels
	periodic_params_n3 = pd.DataFrame(periodic_params_n3)

	for i in range(len(peak_labels)):
		 peak_no = periodic_params_n3.columns[i * 3: (i + 1) * 3]
		 heading = peak_labels[i]
		 periodic_params_n3.rename(columns={col: heading for col in peak_no},
										inplace=True)  

	# make a df of aperiodic_params and add parameter labels
	aperiodic_params_n3 = pd.DataFrame(aperiodic_params_n3)

	aperiodic_params_n3.columns=['Exponent','Offset','R^2','Error'] 

	# compiling data into a single wake dataframe
	report_n3=pd.merge(aperiodic_params_n3, periodic_params_n3,  left_index= True, right_index= True)

	# add epoch_channel labels
	report_n3.insert(loc=0, column='Stage', value='N3')
	report_n3.insert(1, 'Channel',channel_n3)
	report_n3.insert(2, 'Epoch',epoch_n3)

	#%% REM SLEEPSTAGE

	# initializing empty lists for storing  parameters 
	epoch_rem=[] 
	channel_rem=[]
	aperiodic_params_rem=[]
	periodic_params_rem=[]

	# looping over epochs and channel,outcome- epoch x channel no of data entries, here 190
	for epoch in range (20,30):
		 for channel in range(psd_rem.shape[1]):

			  temp_periodic = [] # stores periodic params temporarily

			  # fitting spectra
			  fm.fit(freqs, psd_rem[epoch, channel, :])

			  # get results
			  res_rem = fm.get_results()

			  # append aperiodic vals to a list
			  aperiodic_params_rem.append([res_rem.aperiodic_params[0], 
				                           res_rem.aperiodic_params[1],
				                           res_rem.r_squared,
				                           res_rem.error])

			  # updating epoch x channel vals
			  epoch_rem.append (f"rem_Epoch_{epoch+1}")
			  channel_rem.append(f"Channel_{channel+1}")

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

					# plot the fitted spectra, assign results to fm 
					fm.results=res_rem  
					fm.plot(plot_aperiodic=True, plot_peaks='shade', 
					        peak_kwargs={'color' : 'green'}, plt_log=False)
					plt.title(f"rem, Epoch {epoch+1}, channel {channel+1}")
					plt.show()
					plt.savefig('/serverdata/ccshome/nandanik/Documents/results/' 
					            + 'rem_' + 'Epoch_' + str(epoch) + '_' 
					            + channels_to_pick[channel],
					            dpi = 600)
					plt.close()

	# make df of periodic_params and add peak labels
	periodic_params_rem = pd.DataFrame(periodic_params_rem)

	for i in range(len(peak_labels)):
		 peak_no = periodic_params_rem.columns[i * 3: (i + 1) * 3]
		 heading = peak_labels[i]
		 periodic_params_rem.rename(columns={col: heading for col in peak_no},
										inplace=True)

	# make a df of aperiodic_params and add parameter labels
	aperiodic_params_rem = pd.DataFrame(aperiodic_params_rem)

	aperiodic_params_rem.columns=['Exponent','Offset','R^2','Error'] 

	# compiling data into a single wake dataframe
	report_rem=pd.merge(aperiodic_params_rem , periodic_params_rem,  left_index= True, right_index= True)

	# add epoch_channel labels
	report_rem.insert(loc=0, column='Stage', value='REM')
	report_rem.insert(1, 'Channel',channel_rem)
	report_rem.insert(2, 'Epoch',epoch_rem)

	#%% COMPILE SLEEP DATAFRAME 

	#compile dataframe
	report_sleepstages=pd.concat([report_w,report_n1,report_n2,report_n3,report_rem], 
	                             axis=0)

	#introducing subject name
	report_sleepstages.insert(loc=0, column='subject', value='subjectname')

	#reset index
	report_sleepstages.reset_index(drop=True, inplace= True)

	#%% EDITTING AND FINALIZING

	#remove entries with r_squred vals <0.9
	report_sleepstages_II= report_sleepstages[report_sleepstages['R^2'] >= 0.9]
	report_sleepstages_II.reset_index(drop=True, inplace= True)

	#%% AVERAGING ACROSS EPOCHS AND TOPOPLOTS

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

	#%% Parameters for Topoplot 

	# Exponet vals

	# make a 2D dataframe containing exponent vals corresponding to 19 channels for 5 sleepstages
	Exponent_vals= Channel_avg_vals.pivot(index='Channel', columns='Stage', values='Exponent')

	# The index MUST be the channel names for yasa 
	Exponent_vals.index= channels_to_pick

	# Offset vals

	# make a 2D dataframe containing offset vals corresponding to 19 channels for 5 sleepstages
	Offset_vals= Channel_avg_vals.pivot(index='Channel', columns='Stage', values='Offset')

	# The index MUST be the channel names for yasa 
	Offset_vals.index= channels_to_pick

	#%% TOPOPLOT

	#define sleep_stages
	sleep_stages=['W','N1','N2','N3','REM']

	#EXPONENT TOPO

	#loop over sleep stages and plot the data
	for i in range(0,len(sleep_stages)):
		 stage= sleep_stages[i]
		 min_val= Exponent_vals.min().min()
		 max_val= Exponent_vals.max().max()
		 yasa.topoplot(Exponent_vals[stage], title =stage,
		                vmin= min_val,
		                vmax=max_val,
		                cmap = 'coolwarm',
		                n_colors= 10 )
		 plt.tight_layout() #adjusts layout of plot
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
		             + 'Exponent_' + stage , facecolor='white')
		 plt.close()
						
	#OFFSET TOPO

	#loop over sleep stages and plot the data
	for i in range(0,len(sleep_stages)):
		 stage= sleep_stages[i]
		 min_val= Offset_vals.min().min()
		 max_val= Offset_vals.max().max()
		 yasa.topoplot(Offset_vals[stage], title =stage, 
		                vmin=min_val,
		                vmax=max_val,
		                cmap = 'coolwarm',
		                n_colors= 10 )       
		 plt.tight_layout() #adjusts layout of plot      
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
		             + 'Offset_' + stage , facecolor='white')
		 plt.close() 
              