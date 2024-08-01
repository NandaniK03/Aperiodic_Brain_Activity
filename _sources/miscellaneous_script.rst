
Miscellaneous Scripts
----------------------

Here are some scripts useful for performing tasks such as categorizing data into groups, acquiring the PSG files details, removing files by index, computing runtime of the program.

**Assigning Subjects to Groups: Topoplots**
 
Categorize subjects into Meditators and Controls.
Make Topoplots for the two groups.
 
 .. code-block:: python

	#%%Import libraies | Block 0

	import pandas as pd
	from scipy.stats import trim_mean
	import yasa
	import matplotlib.pyplot as plt

	#%% Load files | Block 1

	# load the file containing subject info
	masterfile = pd.read_csv('/serverdata/ccshome/nandanik/Downloads/mastersheet.csv')

	# load the fil contaning compiled aperiodic paramters 
	report_sleepstages_II = pd.read_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_aperiodic_fooof_sleepdata_.csv')

	#%% Assign category to subjects | Block 1

	# filenames
	filenames = report_sleepstages_II['Subject'].unique()

	# group labels
	group = []
	for names in filenames:
		 names = names.split('.')[0]
		 index = masterfile[masterfile['MappingCode'] == names].index
		 group.append(masterfile['Group'][index].values[0])
		 
	# Add the group label to 'report_sleepstages_II' dataframe
	report_sleepstages_II['Group'] = None  # Initialize the new column with None values

	for index, names in enumerate(filenames):
		 indices = report_sleepstages_II[report_sleepstages_II['Subject'] == names].index
		 report_sleepstages_II.loc[indices, 'Group'] = group[index]
	 
	# saving to a csv file
	report_sleepstages_II.to_csv('/serverdata/ccshome/nandanik/Documents/CSV/nk_aperiodic_fooof_sleepdata_2.csv', index= False)

	#%% Averaging across epochs | Block 2

	# Trimmed mean
	# Define the trim percentage (here, 10%)
	trim_percentage = 0.1

	#  Group by columns and calculate trimmed mean for each group
	Channel_avg_vals = report_sleepstages_II.groupby(['Stage', 'Channel', 'Group']).apply(lambda group: group.iloc[:, 2:].apply(trim_mean, proportiontocut=trim_percentage))

	#reset the index to convert the grouped columns ('Channel' and 'Stage') back to regular columns
	Channel_avg_vals = Channel_avg_vals.reset_index()

	# Extracting the channel number from the 'Channel' column
	Channel_avg_vals['Channel'] = Channel_avg_vals['Channel'].str.split('_').str[1].astype(int)

	# Sorting the DataFrame by 'Channel' and 'Stage'
	Channel_avg_vals = Channel_avg_vals.sort_values(by=['Stage','Channel'], ascending=[True, True])
	Channel_avg_vals.reset_index(drop= True, inplace= True)

	# split the dataframe into controls and meditators
	Med_group = Channel_avg_vals[Channel_avg_vals['Group'] == 'MED']
	Cnt_group = Channel_avg_vals[Channel_avg_vals['Group'] == 'CNT'] 

	#%% Parameters for Topoplot | Block 3

	channels_to_pick_topo = ['Fp1', 'Fp2', 'F3', 'F4', 'C3', 'C4', 'P3', 'P4', 'O1', 'O2',
		  'F7', 'F8', 'T3', 'T4', 'Fz', 'Cz', 'Pz', 'A1', 'A2']

	# Exponet vals

	# make a 2D dataframe containing exponent vals corresponding to 19 channels for 5 sleepstages
	Exponent_med = Med_group.pivot(index='Channel', columns='Stage', values='Exponent')
	Exponent_cnt = Cnt_group.pivot(index='Channel', columns='Stage', values='Exponent')

	# The index MUST be the channel names for yasa
	Exponent_med.index = channels_to_pick_topo
	Exponent_cnt.index = channels_to_pick_topo

	# Offset vals

	# make a 2D dataframe containing offset vals corresponding to 19 channels for 5 sleepstages
	Offset_med = Med_group.pivot(index='Channel', columns='Stage', values='Offset')
	Offset_cnt = Cnt_group.pivot(index='Channel', columns='Stage', values='Offset')

	# The index MUST be the channel names for yasa
	Offset_med.index = channels_to_pick_topo
	Offset_cnt.index = channels_to_pick_topo

	#%% TOPOPLOT | Block 4

	#define sleep_stages
	sleep_stages = ['W','N1','N2','N3','REM']

	# MEDITATORS

	#EXPONENT TOPO
	#loop over sleep stages and plot the data# Create a 3-D array
	for i in range(0,len(sleep_stages)):
		 vmax = Channel_avg_vals['Exponent'].max()
		 vmin = Channel_avg_vals['Exponent'].min()
		 stage = sleep_stages[i]
		 yasa.topoplot(Exponent_med[stage], title =stage,
							vmin= vmin,
							vmax= vmax,
							cmap = 'coolwarm',
							n_colors= 10 )
		 plt.tight_layout() #adjusts layout of plot
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
						 + 'Exponent_MED_' + stage , facecolor='white')
		 plt.close()

	#OFFSET TOPO
	#loop over sleep stages and plot the data
	for i in range(0,len(sleep_stages)):
		 vmax = Channel_avg_vals['Offset'].max()
		 vmin = Channel_avg_vals['Offset'].min()
		 stage = sleep_stages[i]
		 yasa.topoplot(Offset_med[stage], title =stage,
							vmin= vmin,
							vmax= vmax,
							cmap = 'coolwarm',
							n_colors= 10 )
		 plt.tight_layout() #adjusts layout of plot
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
						 + 'Offset_MED_' + stage , facecolor='white')
		 plt.close()

	# CONTROLS

	#EXPONENT TOPO
	#loop over sleep stages and plot the data# Create a 3-D array
	for i in range(0,len(sleep_stages)):
		 vmax = Channel_avg_vals['Exponent'].max()
		 vmin = Channel_avg_vals['Exponent'].min()
		 stage = sleep_stages[i]
		 yasa.topoplot(Exponent_cnt[stage], title =stage,
							vmin= vmin,
							vmax= vmax,
							cmap = 'coolwarm',
							n_colors= 10 )
		 plt.tight_layout() #adjusts layout of plot
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
						 + 'Exponent_CNT_' + stage , facecolor='white')
		 plt.close()

	#OFFSET TOPO
	#loop over sleep stages and plot the data
	for i in range(0,len(sleep_stages)):
		 vmax = Channel_avg_vals['Offset'].max()
		 vmin = Channel_avg_vals['Offset'].min()
		 stage = sleep_stages[i]
		 yasa.topoplot(Offset_cnt[stage], title =stage,
							vmin= vmin,
							vmax= vmax,
							cmap = 'coolwarm',
							n_colors= 10 )
		 plt.tight_layout() #adjusts layout of plot
		 plt.show()
		 plt.savefig('/serverdata/ccshome/nandanik/Documents/Topoplots/'
						 + 'Offset_CNT_' + stage , facecolor='white')
		 plt.close()
 
**PSG file details**

Access PSGfile properties.
Dataframe: channels. srate and psgfilename for each subject.
Sort files and remove files with srate != 500.

.. code-block:: python 

	#%% Load files

	# specify folderpath
	folder_path_psg = '/serverdata/ccshome/nandanik/Documents/FOOOF_data/data'
	file_pattern_psg = '*.edf'  

	# List containing files names
	os.chdir(folder_path_psg)
	psg_files = sorted(gb.glob( file_pattern_psg))

	#%% extract file properties
	n_channel=[]
	channel_names=[]
	sfreq_n=[]
	for files in psg_files:

		 edfdata = mne.io.read_raw_edf(files, preload=True)
		 srate = int(edfdata.info['sfreq'])

		 channels_to_pick = ['Fp1', 'FP1' ,'EEG Fp1','EEG FP1' , 'Fp2', 'FP2', 'EEG Fp2', 'EEG FP2',
		  'F3', 'EEG F3', 'F4', 'EEG F4', 'C3', 'EEG C3', 'C4', 'EEG C4', 'P3', 'EEG P3', 'P4', 
		  'EEG P4', 'O1', 'EEG O1', 'O2', 'EEG O2', 'F7', 'EEG F7', 'F8', 'EEG F8', 'T3', 'EEG T3', 
		  'T4', 'EEG T4', 'Fz', 'FZ' , 'EEG Fz', 'EEG FZ' , 'Cz', 'CZ' , 'EEG Cz', 'EEG CZ', 
		  'Pz', 'PZ','EEG Pz', 'EEG PZ', 'A1', 'A2' , 'EEG A1', 'EEG A2']

		 edfdata.pick_channels(channels_to_pick)
		 
		 num = len(edfdata.ch_names)
		 n_channel.append(num)
		 
		 name = edfdata.ch_names
		 channel_names.append(name)

		 sfreq= edfdata.info['sfreq']
		 sfreq_n.append(sfreq)
		 

	print("Total files processed:", len(psg_files))
	print("Total entries in n_channel:", len(n_channel))

	#%% dataframe
	psg_channel= pd.DataFrame({'file': psg_files,
	                          'n_channel': n_channel,
	                          'channels': channel_names,
	                          'sfreq': sfreq_n})
										
	# files with different sampling frequencies
	count1 = (psg_channel['sfreq'] == 200).sum()
	count2 = (psg_channel['sfreq'] == 500).sum()
	count3 = (psg_channel['n_channel'] == 19).sum()
	
	# files to be deleted
	delete_files= pd.DataFrame()
	delete_files= delete_files.append(psg_channel[psg_channel['sfreq']!=500])

	# files to be kept
	valid_files= pd.DataFrame()
	valid_files= valid_files.append(psg_channel[psg_channel['sfreq']==500])

	#%% plotting data
	edfdata = mne.io.read_raw_edf(files, preload=True)

	edfdata.filter(1,None,fir_design='firwin').load_data()
	edfdata.filter(None,40,fir_design='firwin').load_data()

	edfdata.plot()



**Remove unwanted PSG files by index**

.. code-block:: python

	# Indices of files to remove
	psg_del = [ index1, index2, index3, .....]  

	#remove unwanted psg files 
	psg_files_all = [x for i, x in enumerate(psg_files_all) if i not in psg_del]

	# Indices of files to remove
	scored_del = [index1, index2, index3, .....]

	#remove unwanted scored files
	scored_files_all = [x for i, x in enumerate(scored_files_all) if i not in scored_del]

**Program Runtime**

.. code-block:: python

	import time

	start_time = time.time()

	# this is where your loop goes         

	end_time = time.time()
	duration = end_time - start_time

	print("Loop duration:", duration, "seconds")

