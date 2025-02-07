# Low Mag to High Mag Pipeline - Zanetti Lab 2025 

Hi guys! This doc should contain all the information you need to make low mag tomos and use them to inform the positioning of your high mag tomos. :) 

## Useful Info
For sbatch commands

Partitions we use: 
| Partition     | GPU           | CPU   | CPU per GPU |
| :-------------: | :-------------:| :-----:| :-----------:|
| ga100         | 4             | 256   | 64          |
| gl40          | 4             | 64    | 16          |

ga100, has 4 GPUs, 256 CPUs
gl140, has 4 GPUs, 64 CPUs

You can use either partition, just change the --partition line and add a --reservation=gl40 line to use gl40.  
If you are requesting n GPUs, request n/4 CPU.

# Reconstruct the Low Mag Tomos

cp files from krios1 to your processing directory. 
mkdir motioncorr

**1. Extract tilts from mdocs**
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=tilts
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=tlt_err.log
#SBATCH --output=tlt_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0

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
       temp_list_file="${base_name}_list.txt"

       # Create the list file (though the script doesn't actually use this file, just creates it)
       echo "Creating list file: $temp_list_file"

       # Run the extracttilts command
       extracttilts -other "$mdoc_file" -output "${base_name}.rawtlt" -tilts

       # Remove the temporary list file
       rm "$temp_list_file"
   else
       echo "File does not exist: $mdoc_file"
   fi
done
~                                                                                                                                                                                                                      
```
**2. Imod motion correction**
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=motioncorr
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=motioncorr_err.log
#SBATCH --output=motioncorr.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0

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
       alignframes -evenodd 2  -list "$temp_list_file" -refine 3 -volt 300 -gpu 3 -reorder 1 -pixel 7.3 -output "./motioncorr/${base_name}_aligned.mrc" -tilt "${base_name}.rawtlt"

       # Remove the temporary list file
       rm "$temp_list_file"
   else
       echo "File does not exist: $mdoc_file"
   fi
done
~
```
cd motioncorr

**3. Extract tilts from motion corrected mrcs**
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=tilts
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=64
#SBATCH --error=tlt_err.log
#SBATCH --output=tlt_out.log
#SBATCH --gres=gpu:1
#SBATCH --mem=0
#SBATCH --time=1-0:0:0

input_folder="."
mdoc_files=($input_folder/*.mrc)

# Check if there are any .mrc files
if [ ${#mdoc_files[@]} -eq 0 ]; then
   echo "No .mrc files found in $input_folder"
   exit 1
fi

# Loop through each mrc file
for mdoc_file in "${mdoc_files[@]}"; do
   if [ -e "$mdoc_file" ]; then
       base_name=$(basename "$mdoc_file" .mrc)
       temp_list_file="${base_name}_list.txt"

       # Create the list file (though the script doesn't actually use this file, just creates it)
       echo "Creating list file: $temp_list_file"

       # Run the extracttilts command
       extracttilts -input "$mdoc_file" -output "${base_name}.rawtlt" -tilts

       # Remove the temporary list file
       rm "$temp_list_file"
   else
       echo "File does not exist: $mdoc_file"
   fi
done
~                                                                                                                                                                                                         
```
mkdir output 

**4. Run AreTomo align and reconstruct**
```
#!/bin/bash
#SBATCH --partition=ga100
#SBATCH --job-name=AreTomo
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --cpus-per-task=256
#SBATCH --error=AT_err.log
#SBATCH --output=AT_out.log
#SBATCH --gres=gpu:4
#SBATCH --mem=0
#SBATCH --time=1-0:0:0

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
       AreTomo -InMrc "$mrc_file" -OutMrc ./output/"${base_name}_rec_tilt.mrc" -PixSize 7.3 -AngFile "${base_name}.rawtlt" -OutBin 2 -FlipVol 1 -TiltCor 1 20
   else
       echo "File does not exist: $mrc_file"
   fi
done
```
**5. Inspect Low Mag Tomos**
Look for ERES-like areas. 

# Setting up High Mag Positions
Ask Andrea to please change the settings from low to high mag. 

1. In Tomo navigate to the FOV of the low mag tomo in the search map. 
2. Open the low mag tomo. Open slicer, rotate 90 degrees and then mentally flip the image. 
3. Use your brain to correlate between the tomo and the search map. 
4. Add high mag positions. 
   * Naming rules L_LamellaNumber_Low_LowTomoNumber_High_HighTomoNumber
5. Update defocus 7-10. 



