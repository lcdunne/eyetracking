# How to interface PsychoPy with Tobii

[Alexander Nitka has a tutorial](https://github.com/aleksandernitka/EyeTracking_Tobii_PsychoPy_IOHUB) which was extremely useful for me in setting this up (thanks!). I have updated his code as I think there were some PsychoPy updates preventing it from working for me. Also, since his processing code was written in R, I decided to write a processing script in Python. I hope that this is useful.

### Required Settings

Before the following works, you should add an experiment info field called `eyetracking`, which should be set to either 1 or 0. As well as this, in your experiment conditions file you should have a column that specifies your condition with a unique number. If this is not a number, you will need to update the code. This column should be called `condition_number`. If this is called something else you will need to update the code. 

### Step 1: Insert a code component

We can do this almost entirely with code components which is nice. First, create a code component in the trial where your stimuli are presented.

### Step 2: Paste into `Begin Experiment` tab

In the `Begin Experiment` tab insert the following code:


```python
# Code based on Alexander Nitka's tutorial from the GitHub link
import os
import shutil

if expInfo['eyetracking']:
    try:
        from psychopy.iohub import launchHubServer
        from psychopy.core import getTime, wait
        
        iohub_config = {'experiment_code':'my_experiment', # placeholder to enable file outputting
        'session_code':f"et_{expInfo['participant']}_{expInfo['date']}",
        'eyetracker.hw.tobii.EyeTracker':{'name': 'tracker', 'runtime_settings': {'sampling_rate': 120}}}
        io = launchHubServer(**iohub_config) # We can send messages to io which get stored in the hdf5 file.

        # Get the eye tracker device.
        tracker = io.devices.tracker
        
        ## optionally run eyetracker calibration
        #r = tracker.runSetupProcedure()
        
        # Try not to change this, otherwise the processing script needs updating
        # These have times associated with them as well: events['text'] and events['time']
        io.sendMessageEvent(text=f"EXPERIMENT_NAME: {expInfo['expName']}")
        io.sendMessageEvent(text=f"EXPERIMENT_DATE: {expInfo['date']}")
        io.sendMessageEvent(text=f"SUBJECT_ID: {expInfo['participant']}")
        
    except Exception as e:
       print("!! Error starting ioHub: ",e," Exiting...")
       sys.exit(1)
```

### Step 3: Paste into `End Experiment` tab


```python
if expInfo['eyetracking']:
    tracker.setConnectionState(False)
    io.sendMessageEvent(text="EXPERIMENT FINISHED")
    io.quit()
    
    old_filename = f"{iohub_config['session_code']}.hdf5"
    new_filename = f"eyetracking_data//{old_filename}"
    
    if not os.path.exists('eyetracking_data'):
        os.mkdir('eyetracking_data') # Make an output directory
    
    os.rename(old_filename, new_filename) # This just moves it to the new directory.
```

### Step 4: Paste into `Begin Routine` tab


```python
if expInfo['eyetracking']:
    io.clearEvents('all')
    tracker.setRecordingState(True)
    ## Send some info about the experiment to the IOHUB
    io.sendMessageEvent(text=f"start_TRIAL-{trials.thisN}_CONDITION-{condition_number}") # start of the trial
```

### Step 5: Paste into `End Routine` tab


```python
if expInfo['eyetracking']:
    tracker.setRecordingState(False)
    io.sendMessageEvent(text=f"end_TRIAL-{trials.thisN}_CONDITION-{condition_number}") # end of the trial
```

### Step 6: Run this code

Alexander Nitka wrote some R code that preprocesses the data. In a similar way, here is a Python script that does this. In order to use this you need the `pandas` package (I think it might come with PsychoPy).


```python
import h5py
import pandas as pd

filepath = r'C:\Users\L\Google Drive\PhD\experiments\ATTMEM\Attmem_PsychoPy\eyetracking_data'
fname = 'et_99_2020_Mar_11_1522.hdf5'
f = h5py.File(f'{filepath}//{fname}', 'r')
events = f['/data_collection/events/experiment/MessageEvent']
eyetr = f['/data_collection/events/eyetracker/BinocularEyeSampleEvent']

# Create a dataframe from all write events
wedf = pd.DataFrame(events[:])
# Replace the text column with decoded text labels
wedf['text'] = wedf['text'].apply(lambda x: x.decode())
tedf = wedf.loc[(wedf['text'].str.contains('start')) | (wedf['text'].str.contains('end'))].reset_index(drop=True)
_temp = tedf['text'].str.split('_', expand=True).rename(columns={0:'time_loc', 1:'trial', 2:'condition'})
tedf = pd.concat([tedf, _temp], axis=1)
# Now sort out trial number and condition by splitting on the hyphen
tedf['trial'] = tedf['trial'].str.split('-', expand=True)[1].astype(int)
tedf['condition'] = tedf['condition'].str.split('-', expand=True)[1].astype(int) # if using a condition label then this will give an error - just lose the `.astype(int)`

# events['text'] contains all the events logged in psychopy (E.g. condition nums)
# Events are stored as bytes so convert all to str format
all_event_info = [i.decode() for i in events['text']]

exp_name = all_event_info[0].split(': ')[-1]
exp_date = all_event_info[1].split(': ')[-1]
subject_id = all_event_info[2].split(': ')[-1]

tedf['exp_name'] = exp_name
tedf['exp_date'] = all_event_info[1].split(': ')[-1]
tedf['subject_id'] = all_event_info[2].split(': ')[-1]

etdf = pd.DataFrame(eyetr[:])

# Unstack the dataframe so we have one event per row.
tedf_wide = tedf.set_index(['time_loc', 'trial', 'condition']).unstack(0)
tedf_wide.columns = [f'{i}_{j}' for i, j in tedf_wide.columns]
tedf_wide = tedf_wide.reset_index()

# Segment the eyetracking data based on the start/end of each trial event.
# This will work but might not be most efficient, although we typically won't have too many rows to loop over.
cols_to_get = ['trial', 'condition', 'device_time_start', 'logged_time_start', 'device_time_end', 'logged_time_end', 'time_start', 'time_end']
for r, row in tedf_wide.iterrows():
    conditional_select = (etdf['time']>=row['time_start']) & (etdf['time']<=row['time_end'])
    # Loop over cols to get
    for c in cols_to_get:
        etdf.loc[conditional_select, c] = row[c]

# Compute mean of x and y coords across both eyes
etdf['gaze_x'] = etdf[['left_gaze_x', 'right_gaze_x']].mean(axis=1)
etdf['gaze_y'] = etdf[['left_gaze_y', 'right_gaze_y']].mean(axis=1)
etdf['eyes_x'] = etdf[['left_eye_cam_x', 'right_eye_cam_x']].mean(axis=1)
etdf['eyes_y'] = etdf[['left_eye_cam_y', 'right_eye_cam_y']].mean(axis=1)
etdf['eyes_z'] = etdf[['left_eye_cam_z', 'right_eye_cam_z']].mean(axis=1)
etdf['pupils'] = etdf[['left_pupil_measure1', 'right_pupil_measure1']].mean(axis=1) # x_pupil_measure2 is useless apparently?

# Output the file as a csv
etdf.to_csv(f"{filepath}//{fname.split('.')[0]}.csv", index=False)

f.close()
```
