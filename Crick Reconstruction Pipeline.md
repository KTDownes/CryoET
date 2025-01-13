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
