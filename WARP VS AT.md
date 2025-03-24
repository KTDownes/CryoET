# Directly Comparing WARP and AreTomo
Hey Giulia, here is all the commands I used to create the 27FEB25 AT and WARP outputs.

## AreTomo Pipeline 
Ln frames and mdocs to a processing directory

extracttilts.csh
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
   else
       echo "File does not exist: $mdoc_file"
   endif
end
```
extracts tilts based on mdoc. 

mkdir motioncorr
align_movie.bash
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=motioncorr
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=motioncorr_err.log
#SBATCH --output=motioncorr.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml imod
source $IMOD_DIR/IMOD-linux_vis.sh
input_folder="."
mdoc_files=($input_folder/*.mdoc)

# Check if there are any .mdoc files
if [ ${#mdoc_files[@]} -eq 0 ]; then
   echo "No .mdoc files found in $input_folder"
   exit 1
fi

# Loop through each mdoc file
for mdoc_file in "${mdoc_files[@]}"; do
   if [ -e "$mdoc_file" ]; then
       base_name=$(basename "$mdoc_file" .mdoc)
       eer_files=($input_folder/${base_name}_0*fractions.mrc)

       # Check if there are any fraction files
       if [ ${#eer_files[@]} -eq 0 ]; then
           echo "No fraction files found for $mdoc_file"
           continue
       fi

       # Create the temporary list file
       temp_list_file="${base_name}_fraction_list.txt"
       echo "Creating list file: $temp_list_file"
       > "$temp_list_file" # Clear the file before appending

       # Add all the fraction files to the list file
       for eer_file in "${eer_files[@]}"; do
           echo "$eer_file" >> "$temp_list_file"
       done

       # Process the fraction files
       echo "Processing all fraction files for: $base_name"
       alignframes -evenodd 1  -list "$temp_list_file" -refine 3 -volt 300 -gpu 3 -reorder 1 -pixel 2.24 -output "./motioncorr/${base_name}_aligned.mrc" -tilt "${base_name}.rawtlt"
         
       # Remove the temporary list file
       rm "$temp_list_file"   
   else 
       echo "File does not exist: $mdoc_file"
   fi        
done
~    
```
This outputs motion corrected mrcs to a your folder called motioncorr. The flag evenodd 1 generates half maps. 

cd into motioncorr. 
extracttilts2.csh
```
#!/bin/csh -f
  
set input_folder = "."
set mdoc_files = ($input_folder/*.mrc)
if ( $#mdoc_files == 0 ) then
   echo "No .mdoc files found in $input_folder"
   exit 1
endif
foreach mdoc_file ($mdoc_files)
   if ( -e $mdoc_file ) then
       set base_name = `basename $mdoc_file .mdoc`
       set temp_list_file = "${base_name}_list.txt"
       echo "Creating list file: $temp_list_file"
       extracttilts -input "$mdoc_file" -output "${base_name}.rawtlt" -tilts
   else
       echo "File does not exist: $mdoc_file"
   endif
end
```
extracts tilts based on the corrected mrc. 

mkdir output10A

AreTomo_rec_tilt_10.bsh 
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=AreTomo
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=AT10_err.log
#SBATCH --output=AT10_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=200G
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml AreTomo

input_folder="."
mrc_files=($input_folder/*aligned.mrc)

# Check if there are any .mrc files
if [ ${#mrc_files[@]} -eq 0 ]; then
   echo "No .mrc files found in $input_folder"
   exit 1
fi

# Loop through each mrc file
for mrc_file in "${mrc_files[@]}"; do
   if [ -e "$mrc_file" ]; then
       base_name=$(basename "$mrc_file" .mrc)

       # Run the AreTomo command
       AreTomo -InMrc "$mrc_file" -OutMrc ./output10A/"${base_name}_rec_tilt.mrc" -PixSize 2.24  -AngFile "${base_name}.rawtlt" -OutBin 4.6 -FlipVol 1 -TiltCor 1 -VolZ 2000 -AlnZ 600
   else
       echo "File does not exist: $mrc_file"
   fi
done
```
## WARP pipeline
mkdir WARP and cd inside  
mkdir original mdocs ln mdocs here  
mkdir frames ln frames here  
cd original mdocs and add adjust_angles.py inside  
adjust_angles.py  
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
~                                                                                                                                                                                                                   
~
```                                                             
corrects for pretilt of lamella. 

mkdir WARP/editted_mdocs
mv test*mdocs to editted mdocs 

Inside WARP directory:
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
```
```
WarpTools create_settings \
--output warp_tiltseries.settings \
--folder_processing warp_tiltseries \
--folder_data tomostar \
--extension "*.tomostar" \
--angpix 2.24 \  #change to suit your data
--exposure 3.5 \  #change to suit your data
--tomo_dimensions 3710x3838x2000 #change to suit your data
```
Creates the settings to run WARP. 2000 is the final reconstruction volume. In AT this is denoted in VolZ 2000. 

motioncorr_ctf
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

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
  --device_list 3
~                 
```
Import can be run by command line. 
```
ml WarpTools/2.0.0
source activate warp
# TS Import
  WarpTools ts_import \
  --mdocs mdocs \
  --frameseries warp_frameseries \
  --tilt_exposure 3.5 \
  --dont_invert \
  --output tomostar/
  --min_intensity 0.7 \
```
Stack
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=16
#SBATCH --error=stack_err.log
#SBATCH --output=stack_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=200G
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp
WarpTools ts_stack \
  --settings warp_tiltseries.settings \
  --angpix 10
```
Makes motion corrected stacks. Found in warp_tiltseries

AreTomo.bsh
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP_AT2
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=4
#SBATCH --error=AT2_err.log
#SBATCH --output=AT2_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=200G
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp
ml AreTomo2
WarpTools ts_aretomo --settings warp_tiltseries.settings --angpix 10 --alignz 600 --axis 93 --axis_iter 3 --device_list 4 --min_fov 0.1
```
The flag -- min_fov excludes tilts which have drifted away from the tomograms field of view. You can determine how much shift is acceptable, I use 0.1 which disables tilts containing less than 10% of the tomos field of view.  
  
**Difference to stand alone AT:**  
    Our tilt axis is approx 93. axis_iter searches around this value to find best axis. In standalone AT this is just denoted as TiltCor1 (find and correct my tilt axis).   
    Standalone has a default 0.7 DarkTol flag. This removes dark tilts. Cannot run this through warp. Removal of dark tilts occurs in the import step with min_intensity. Not sure of the relationship between DarkTol and min_intensity. I found min_intensity combined with the min_fov flag removes similar (sometimes more) tilts than AT and results in better recons than if neither are flag or if each are flagged independently.   

Sbatch defocus hand and flipping if needed  
Warp_hand
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=16
#SBATCH --error=defocus_err.log
#SBATCH --output=defocus_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=200G
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp
WarpTools ts_defocus_hand --settings warp_tiltseries.settings --check
```
```
Warp_flip
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=16
#SBATCH --error=warp_err.log
#SBATCH --output=warp_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=200G
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp
WarpTools ts_defocus_hand --settings warp_tiltseries.settings --set_flip
```
Warp_ctf
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=ctf_err.log
#SBATCH --output=ctf_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp
WarpTools ts_ctf --settings warp_tiltseries.settings --range_high 8 --defocus_max 11.5 --defocus_min 5.5 
```
Warp_reconstruct 
```
#!/bin/bash
#SBATCH --partition=gl40
#SBATCH --job-name=WARP
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=recon_err.log
#SBATCH --output=recon_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0
#SBATCH --reservation=gl40

ml WarpTools/2.0.0
source activate warp

WarpTools ts_reconstruct --settings warp_tiltseries.settings --angpix 10 --halfmap_frames --dont_invert
```

