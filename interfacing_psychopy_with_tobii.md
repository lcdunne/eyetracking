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
etdf['pupils1'] = etdf[['left_pupil_measure1', 'right_pupil_measure1']].mean(axis=1)
etdf['pupils2'] = etdf[['left_pupil_measure2', 'right_pupil_measure2']].mean(axis=1)

# Output the file as a csv
etdf.to_csv(f"{filepath}//{fname.split('.')[0]}.csv", index=False)

f.close()
```


```python
etdf
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>experiment_id</th>
      <th>session_id</th>
      <th>device_id</th>
      <th>event_id</th>
      <th>type</th>
      <th>device_time</th>
      <th>logged_time</th>
      <th>time</th>
      <th>confidence_interval</th>
      <th>delay</th>
      <th>...</th>
      <th>logged_time_end</th>
      <th>time_start</th>
      <th>time_end</th>
      <th>gaze_x</th>
      <th>gaze_y</th>
      <th>eyes_x</th>
      <th>eyes_y</th>
      <th>eyes_z</th>
      <th>pupils1</th>
      <th>pupils2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>65</td>
      <td>52</td>
      <td>1.152367e+12</td>
      <td>14.182478</td>
      <td>14.168915</td>
      <td>0.0</td>
      <td>0.013563</td>
      <td>...</td>
      <td>20.189254</td>
      <td>13.805807</td>
      <td>20.189104</td>
      <td>-162.941666</td>
      <td>-96.425110</td>
      <td>0.623075</td>
      <td>0.381974</td>
      <td>0.348952</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>66</td>
      <td>52</td>
      <td>1.152367e+12</td>
      <td>14.185055</td>
      <td>14.177663</td>
      <td>0.0</td>
      <td>0.007392</td>
      <td>...</td>
      <td>20.189254</td>
      <td>13.805807</td>
      <td>20.189104</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>67</td>
      <td>52</td>
      <td>1.152367e+12</td>
      <td>14.192105</td>
      <td>14.185805</td>
      <td>0.0</td>
      <td>0.006300</td>
      <td>...</td>
      <td>20.189254</td>
      <td>13.805807</td>
      <td>20.189104</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>68</td>
      <td>52</td>
      <td>1.152367e+12</td>
      <td>14.204327</td>
      <td>14.194054</td>
      <td>0.0</td>
      <td>0.010273</td>
      <td>...</td>
      <td>20.189254</td>
      <td>13.805807</td>
      <td>20.189104</td>
      <td>-123.767166</td>
      <td>-3.505046</td>
      <td>0.621213</td>
      <td>0.381383</td>
      <td>0.530113</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>69</td>
      <td>52</td>
      <td>1.152367e+12</td>
      <td>14.211650</td>
      <td>14.202451</td>
      <td>0.0</td>
      <td>0.009199</td>
      <td>...</td>
      <td>20.189254</td>
      <td>13.805807</td>
      <td>20.189104</td>
      <td>-117.420387</td>
      <td>-46.732365</td>
      <td>0.621200</td>
      <td>0.381337</td>
      <td>0.530228</td>
      <td>3.078796</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>2500</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2579</td>
      <td>52</td>
      <td>1.152390e+12</td>
      <td>37.699714</td>
      <td>37.689376</td>
      <td>0.0</td>
      <td>0.010338</td>
      <td>...</td>
      <td>37.734314</td>
      <td>32.724899</td>
      <td>37.734092</td>
      <td>20.607258</td>
      <td>-31.048328</td>
      <td>0.548459</td>
      <td>0.390005</td>
      <td>0.535358</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2501</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2580</td>
      <td>52</td>
      <td>1.152390e+12</td>
      <td>37.707752</td>
      <td>37.697214</td>
      <td>0.0</td>
      <td>0.010538</td>
      <td>...</td>
      <td>37.734314</td>
      <td>32.724899</td>
      <td>37.734092</td>
      <td>21.896584</td>
      <td>-34.358776</td>
      <td>0.548560</td>
      <td>0.390127</td>
      <td>0.535537</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2502</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2581</td>
      <td>52</td>
      <td>1.152390e+12</td>
      <td>37.716567</td>
      <td>37.705191</td>
      <td>0.0</td>
      <td>0.011376</td>
      <td>...</td>
      <td>37.734314</td>
      <td>32.724899</td>
      <td>37.734092</td>
      <td>7.449989</td>
      <td>-49.349129</td>
      <td>0.548501</td>
      <td>0.390129</td>
      <td>0.535794</td>
      <td>2.755959</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2503</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2582</td>
      <td>52</td>
      <td>1.152390e+12</td>
      <td>37.724026</td>
      <td>37.713557</td>
      <td>0.0</td>
      <td>0.010469</td>
      <td>...</td>
      <td>37.734314</td>
      <td>32.724899</td>
      <td>37.734092</td>
      <td>12.601379</td>
      <td>-53.369961</td>
      <td>0.548470</td>
      <td>0.390153</td>
      <td>0.535741</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2504</th>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2583</td>
      <td>52</td>
      <td>1.152390e+12</td>
      <td>37.732795</td>
      <td>37.722064</td>
      <td>0.0</td>
      <td>0.010731</td>
      <td>...</td>
      <td>37.734314</td>
      <td>32.724899</td>
      <td>37.734092</td>
      <td>19.575317</td>
      <td>-23.043770</td>
      <td>0.548581</td>
      <td>0.390243</td>
      <td>0.535838</td>
      <td>-1.000000</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
<p>2505 rows × 65 columns</p>
</div>


