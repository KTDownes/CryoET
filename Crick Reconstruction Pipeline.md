Welcome to my cryoET reconstruction pipeline! 

# For fast (ish) and beautiful (ish) tomograms using OnDemand
Request X GPU
I've only tried this with Fluxbox (Xfce is much better for your eyes). 

cd or mkdir a processing directory for your tomograms. Inside cp your mdocs and fractions.mrc. Inside this directory make a directory called motioncorr.
```
ml imod 
source activate $IMOD_DIR/IMOD-linux_vis.sh
```

Create a new csh file caled extracttilts containing: 
```
#!/bin/csh -f
  
set input_folder = "."
set mdoc_files = ($input_folder/*.mdoc)
if ( $#mdoc_files == 0 ) then
   echo "No .mdoc files found in $input_folder"
   exit 1
endif
foreach mdoc_file ($mdoc_files)
   if ( -e $mdoc_file ) then
       set base_name = `basename $mdoc_file .mdoc`
       set temp_list_file = "${base_name}_list.txt"
       echo "Creating list file: $temp_list_file"
       extracttilts -other "$mdoc_file" -output "${base_name}.rawtlt" -tilts
       rm $temp_list_file
   else
       echo "File does not exist: $mdoc_file"
   endif
end
~                                                                                                                                                                                                                                                                             
```
Run this command via:
```
csh -f extracttilts.csh 
```
This will extract the raw tilts from the mdocs and create a .rawtlt file. 

Create a new csh file called align_movie.csh containing: 
```
#!/bin/csh -f
  
set input_folder = "."
set mdoc_files = ($input_folder/*.mdoc)
if ( $#mdoc_files == 0 ) then
   echo "No .mdoc files found in $input_folder"
   exit 1
endif
foreach mdoc_file ($mdoc_files)
   if ( -e $mdoc_file ) then
       set base_name = `basename $mdoc_file .mdoc`
       set eer_files = ($input_folder/${base_name}_0*fractions.mrc)
       if ( $#eer_files == 0 ) then
           echo "No fraction files found for $mdoc_file"
           continue
       endif
       set temp_list_file = "${base_name}_fraction_list.txt"
       echo "Creating list file: $temp_list_file"
       foreach eer_file ($eer_files)
           echo $eer_file >> $temp_list_file
       end
       echo "Processing all fraction  files for: $base_name"
       alignframes -evenodd 2  -list "$temp_list_file"  -refine 3 -volt 300 -gpu 0 -reorder 1 -pixel 2.24  -output "./motioncorr/${base_name}_aligned.mrc" -tilt "${base_name}.rawtlt"
       rm $temp_list_file
   else
       echo "File does not exist: $mdoc_file"
   endif
end
~             
```
Run this command via:
```
csh -f align_movies.csh 
```
This runs the imod command alignframes which generates motion corrected tilt series which can be found in ./motioncorr. The flag -evenodd 2 means even and odd tilt series are compiled which are required for cryocare denoising. 

cd into motioncorr. You can use imod to check the outputed files (i.e. the tilts are in order). 
Inside motioncorr create a file called extracttilts_2.csh:

```
#!/bin/csh -f
  
set input_folder = "."
set mrc_files = ($input_folder/*.mrc)
if ( $#mrc_files == 0 ) then
   echo "No .mrc files found in $input_folder"
   exit 1
endif
foreach mrc_file ($mrc_files)
   if ( -e $mrc_file ) then
       set base_name = `basename $mrc_file .mrc`
       set temp_list_file = "${base_name}_list.txt"
       echo "Creating list file: $temp_list_file"
       extracttilts -input "$mrc_file" -output "${base_name}.rawtlt" -tilts
       rm $temp_list_file
   else
       echo "File does not exist: $mrc_file"
   endif
end
```
Run this command via:
```
csh -f extracttilts_2.csh 
```
This runs the imod command extract tilts but this time from the motion corrected tilt series and will generate another .rawtlt file. The two .rawtlt files are different and yes this is annoying. 

Create a file called AreTomo_rec.csh: 
```
#!/bin/csh -f
  
set input_folder = "."
set mrc_files = ($input_folder/*aligned.mrc)
if ( $#mrc_files == 0 ) then
   echo "No .mrc files found in $input_folder"
   exit 1
endif
foreach mrc_file ($mrc_files)
   if ( -e $mrc_file ) then
       set base_name = `basename $mrc_file .mrc`

       AreTomo -InMrc "$mrc_file" -OutMrc ./output/"${base_name}_rec_2000_1500.mrc" -PixSize 2.24 -AngFile "${base_name}.rawtlt" -OutBin 6 -FlipVol 1 -TiltCor 1 -VolZ 2000 -AlnZ 1500
       AreTomo -InMrc "$mrc_file" -OutMrc ./output/"${base_name}_rec_2000_1000.mrc" -PixSize 2.24 -AngFile "${base_name}.rawtlt" -OutBin 6 -FlipVol 1 -TiltCor 1 -VolZ 2000 -AlnZ 1000

   else
       echo "File does not exist: $mrc_file"
   endif
end
~       
```
Here you can trial several different AreTomo alignment and reconstruction parameters for all your data. If pushed for time, just do this on one example tomogram and use the outputs to inform you as to which conditions work best for your data. However, just because something works for one tomogram does not mean it will work for all. Yes this is sad. So I tend to run a few different conditions on all my tomograms and then come back in the morning and compare them. Good things to change: 
AlnZ
VolZ

For my lamella data the above conditions work quite well. 
For information on which each flag does see: 

https://gensoft.pasteur.fr/docs/AreTomo/1.3.4/AreTomoManual_1.3.0_09292022.pdf

mkdir called output 

Run this command via:
```
csh -f AreTomo_rec.csh 
```
Use imod to look at your beautiful quick tomograms. Best assessed with the XYZ viewer and a cup of tea. 

Select your tomograms which have your feature of interest in them for denoising. 

## ITS WARP TIME
Create a new directory and inside this make two directories, one called frames and the other called original_mdocs, cp your frames and mdocs of your chosen tomograms to the indicated directories. 
Inside original_mdocs create a python script called adjust_angles: 

```
import glob

#Input
mdoc_files = '*.mdoc'
add_angle = +10 #Pretilt to add to tilts
output_prefix = 'test_' #Prefix for the out file name
###

mdoc_files = glob.glob(mdoc_files)

for mdoc_file in mdoc_files:
    filedata = open(mdoc_file, 'r')
    with open(f'{output_prefix}{mdoc_file}', 'a+') as newfile:
        for line in filedata.readlines():
            if 'TiltAngle =' not in line:
                newfile.write(line)
            else:
                original_angle = float(line.split('=',1)[1])
                new_angle = original_angle + add_angle
                newfile.write(f'TiltAngle = {str(new_angle)}\n')
```
This will output mdoc files called test_mdocs. Move these to a directory called mdocs.

In this directory create the following python file - clean_mdoc.py. 
In this directory create an excel file with three columns containing the mdoc file name, the first good tilt number and the last good tilt number. clean_mdoc.py will remove the tilts outside of this range for you. 

Also create a directory called mdoc_mod

```
import os
import re
import pandas as pd
import numpy as np

def process_mdoc_file(input_file, output_file, instance_range):
    # Define the TiltAngle range
    tilt_angles = np.linspace(-60, 60, 41)  # 41 TiltAngles from -60 to 60

    print(f"Processing file: {input_file}")

    # Read the mdoc file
    if not os.path.exists(input_file):
        print(f"Warning: File '{input_file}' not found, skipping...")
        return

    with open(input_file, 'r') as file:
        lines = file.readlines()

    output_lines = []
    keep_section = False
    current_tilt_angle = None

    # Loop through the lines in the mdoc file
    for i, line in enumerate(lines):
        # Add header lines to the output before the first [ZValue = x]
        if '[ZValue' in line and not output_lines:
            output_lines.extend(lines[:i])  # Keep everything before the first [ZValue]

        # Check for the [ZValue] header which marks the start of a section
        if line.startswith('[ZValue'):
            # Try to extract the TiltAngle from the next line (e.g., TiltAngle = -54)
            tilt_angle_match = re.search(r'TiltAngle = (-?\d+\.\d+)', lines[i+1])

            if tilt_angle_match:
                tilt_angle = float(tilt_angle_match.group(1))

                # Find the corresponding instance for the TiltAngle (mapping it to 1-41)
                # We approximate the tilt_angle to the closest value in the tilt_angles array
                instance = np.argmin(np.abs(tilt_angles - tilt_angle)) + 1  # +1 to match 1-based indexing

                # Print the tilt angle and instance number outside the if condition
                print(f"Found TiltAngle: {tilt_angle} -> Corresponding instance: {instance}")
                print(f"Searching for TiltAngle {tilt_angle} (instance {instance})")

                # Determine whether the current section should be kept
                if instance >= instance_range[0] and instance <= instance_range[1]:
                    keep_section = True
                    print(f"Keeping section with TiltAngle: {tilt_angle} (Instance: {instance})")
                else:
                    keep_section = False
                    print(f"Skipping section with TiltAngle: {tilt_angle} (Instance: {instance})")
            else:
                print(f"No TiltAngle found in this section at index {i}")

        # If we should keep the section, add it to the output
        if keep_section:
            output_lines.append(line)

    # Write the processed lines to the output file
    if output_lines:
        with open(output_file, 'w') as file:
            file.writelines(output_lines)
        print(f"File written to {output_file}")
    else:
        print(f"Warning: No sections were kept for '{input_file}'")
          
def main():
    # Read the Excel file with the instance ranges
    excel_file = 'L17_P1_2_Tilts.xlsx'  # Updated file name
    df = pd.read_excel(excel_file, header=None)

    # Process each row in the Excel file
    for _, row in df.iterrows():
        file_id = row[0]
        
        # If file_id is NaN, skip this row
        if pd.isna(file_id): 
            print(f"Warning: Missing file_id at row {_}, skipping...")
            continue
        
#        file_id = int(file_id)
        first_instance = int(row[1])
        last_instance = int(row[2])

        # Generate the input and output file paths
        input_file = f"mdocs/{file_id}"  # Adjust file path as needed
        output_file = f"mdoc_mod/{file_id}"

        # Call the processing function
        process_mdoc_file(input_file, output_file, (first_instance, last_instance))

if __name__ == '__main__':
    main()

```
Lets do some tidying up. rm mdocs and mv mdoc_mod to mdoc. 

Now are mdocs are all happy and correct and ready for warp! 
Some warp commands can be run just from ssh-ing into nemo. However some need GPU and thus must be submitted as sbatch commands until ondemand is working again. 

First step:

```
ml WarpTools/2.0.0
source activate warp

WarpTools create_settings \
--folder_data frames \
--folder_processing warp_frameseries \
--output warp_frameseries.settings \
--extension "*.mrc" \
--angpix 2.24 \  #change to suit your data
--exposure 3.5  #change to suit your data

WarpTools create_settings \
--output warp_tiltseries.settings \
--folder_processing warp_tiltseries \
--folder_data tomostar \
--extension "*.tomostar" \
--angpix 2.24 \  #change to suit your data
--exposure 3.5 \  #change to suit your data
--tomo_dimensions 3710x3838x2000 #change to suit your data
```
Creates all the settings you want. 
Next is your 1st sbatch command: 
I called mine Motioncorr_ctf
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
  WarpTools fs_motion_and_ctf \
  --settings warp_frameseries.settings \
  --m_grid 1x1x4 \
  --c_grid 2x2x1 \
  --c_range_min 40 \
  --c_range_max 8 \
  --c_defocus_min 5.5 \
  --c_defocus_max 11.5 \
  --c_use_sum \
  --out_averages \
  --out_average_halves \
  --perdevice 2 \
  --device_list 0
```
To run sbatch 
sbatch motioncorr_ctf 
“job submitted” 
Squeue -p ga100

You can check the process by looking at the warp_err.log and warp_out.log

Once finished the below jobs can be run in turn. Those requiring sbatch commands can be run in the same manner i.e. sbatch sbatch_file_name. 

```
# TS Import
  WarpTools ts_import \
  --mdocs mdocs \
  --frameseries warp_frameseries \
  --tilt_exposure 3.5 \
  --min_intensity 0.1 \
  --dont_invert \
  --output tomostar/
```

``` 
Stack 
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
WarpTools ts_stack \
  --settings warp_tiltseries.settings \
  --angpix 10
```

```
Warp_aretomo

#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
ml AreTomo2
WarpTools ts_aretomo --settings warp_tiltseries.settings --angpix 10 --alignz 1500 --axis 93 --axis_iter 3 --device_list 4
```
```
Sbatch defocus hand and flipping if needed 
Warp_hand
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
WarpTools ts_defocus_hand --settings warp_tiltseries.settings --check

Warp_flip
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
WarpTools ts_defocus_hand --settings warp_tiltseries.settings --set_flip
```

```
Warp_ctf
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp
WarpTools ts_ctf --settings warp_tiltseries.settings --range_high 8 --defocus_max 11.5 --defocus_min 5.5
```
```                                                                                                 
Warp_reconstruct 
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml WarpTools/2.0.0
source activate warp

WarpTools ts_reconstruct --settings warp_tiltseries.settings --angpix 10 --halfmap_frames --dont_invert

```
Congratulations warp is done. You can now browse your tomograms and check they are ok ahead of denoising. 

# Denoising!

Make even list and odd list wish csh command

```
#!/bin/csh -f
  
set input_folder = "yourpathname/warp_tiltseries/reconstruction"
set even_files = ($input_folder/even/*.mrc)
set odd_files = ($input_folder/odd/*.mrc)
if ( $#even_files == 0 ) then
   echo "No even files found in $input_folder"
   exit 1
endif ($#even_files =! 0 )
    set even_list = "even_list.txt"
    set odd_list = "odd_list.txt"
    echo "Making even and odd files"
endif
foreach even_file ($even_files)
   if ( -e $even_file ) then
       set base_name = `basename $even_file .mrc`
       foreach even_file ($even_files)
            echo '"'$even_file'"', >> $even_list
       end
foreach odd_file ($odd_files)
   if ( -e $odd_file ) then
       set base_name = `basename $odd_file .mrc`
       foreach odd_file ($odd_files)
            echo '"'$odd_file'"', >> $odd_list
   endif
end
```

# CRYOCARE
make a file called train_data_config.json 
Use the csh output to copy and paste the path names of 3 tomograms you want to train on. These should have a variety of your ROI's and cover the range of defocus. 
```
{
    "even": [
        "copy and paste path from the output of the csh/reconstruction/even/High_mag_L12_P1_10.00Apx.mrc",
        "________/reconstruction/even/High_mag_L12_P1_2_10.00Apx.mrc",
        "_________/reconstruction/even/High_mag_L12_P1_3_10.00Apx.mrc"
    ],
    "odd": [
        "_______/reconstruction/odd/High_mag_L12_P1_10.00Apx.mrc",
        "________/reconstruction/odd/High_mag_L12_P1_2_10.00Apx.mrc",
        "________/reconstruction/odd/High_mag_L12_P1_3_10.00Apx.mrc"
    ],
    "patch_shape": [
        72,
        72,
        72
    ],
    "num_slices": 1200,
    "split": 0.9,
    "tilt_axis": "Y",
    "n_normalization_samples": 120,
    "path": "yourpath/warp_tiltseries/reconstruction",
    "overwrite": "True"
}
~
```    
Make sbatch command to run this i.e. cryocare_train_prep

```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=cryocare
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml cryoCARE
source activate cryoCARE-0.3
cryoCARE_extract_train_data.py --conf train_data_config.json
```
                                          
Make a file called train_config.json 

```

{
    "train_data":"yourpath/warp_tiltseries/reconstruction",
    "epochs": 100,
    "steps_per_epoch": 200,
    "batch_size": 16,
    "unet_kern_size": 3,
    "unet_n_depth": 3,
    "unet_n_first": 16,
    "learning_rate": 0.0004,
    "model_name": "denoising_model",
    "path": "yourpath/WARP/warp_tiltseries/reconstruction",
    "overwrite": "True",
    "gpu_id": [
        0
    ]
}

#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=cryocare
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml cryoCARE
source activate cryoCARE-0.3
cryoCARE_train.py --conf train_config.json
*add gpu flag if want to train on >1 gpus
```
This will take some time. Once its complete you can run a prediction on the tomograms you trained your model on to check the success. 

Make a file called predict_config.json and run on your training data 
```
{
    "path": "yourpath/reconstruction/denoising_model.tar.gz",
    "even": [
        "yourpath/reconstruction/even/test_High_mag_L17_P1_2_10.00Apx.mrc"
        ],
    "odd": [
        "yourpath/reconstruction/odd/test_High_mag_L17_P1_2_10.00Apx.mrc"
        ],
    "n_tiles": [
        2,
        2,
        2
    ],
    "output": "yourpath/reconstruction/cryocare_output",
    "overwrite": "True",
    "gpu_id": [
        0
    ]
}
```

Make sbatch file cryocare_predict 
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=cryocare
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml cryoCARE
source activate cryoCARE-0.3
cryoCARE_predict.py --conf predict_config.json

```
This takes a few minutes per tomogram. 
If looks good repeat cryocare_predict command with a new cryocare_predict.json with all tomos.

If you want to push your denoising even futher... 

Next step is isonet. Which fills in the missing wedge. 

# ISONET
Best used with gui, but can make do with sbatch commands. 

Mkdir called isonet 
Inside mkdir called tomos 
Ln tomos want to train on 

```
ml IsoNet 
source activate isonet 

isonet.py prepare_star tomo_folder --pixel_size 10
```

Make mask. Telling isonet where to look for features. i.e. ignore top and bottom 20% of Z.  
```
isonet.py make_mask tomograms.star –z_crop 0.2
```

Sanity check - look at output in mask. If it looks right...

Extract into subtomos 

```
isonet.py prepare_subtomo_star folder_name [--output_star]
```

Now ready to train. Need gpu 

```
sbatch isonet_command 

#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=IsoNet
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=isonet_err.log
#SBATCH --output=isonet_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml IsoNet
source activate isonet
isonet.py refine subtomo.star --iterations 30 --noise_level 0.1,0.2 --noise_start_iter 11,21
```
This takes ages... 

output = results directory. Inside that there is your model. There will be 30 in this case. And 30 iterations of each sub tomo. 

Open all sub tomos of one tomogram and look for a feature in iteration 0. 

Open all iterations of that subtomo to see what isonet does with each iteration. 

Find favourite iteration = this is your trained model. Lets do some predictions! 

Make a new star file with tomograms you want to predict. Suggest start with the training ones to check visually, in this case tomograms.star already exists. So just run below command. 

```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=IsoNet
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=isonet_err.log
#SBATCH --output=isonet_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml IsoNet
source activate isonet
isonet.py predict tomograms.star yourpath/isonet/results/model_iter30.h5 --gpuID 0,1,2,3
```

output = corrected_tomos 
if happy repeat above with all tomos i.e. make tomogram.star with all tomograms using 
```
isonet.py prepare_star tomo_folder --pixel_size 10
```
where tomo_folder contains all your tomograms 

success! 

If you want to segment membranes… 

# Membrain Time
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=membrain
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=isonet_err.log
#SBATCH --output=isonet_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --partition=ga100

ml membrain-seg
source activate Membrane_Venv

membrain segment --tomogram-path /pathtoyourdenoisedtomo --ckpt-path /camp/apps/misc/stp/sbstp/membrain-seg/model/MemBrain_seg_v10_alpha.ckpt --store-connected-components
```




