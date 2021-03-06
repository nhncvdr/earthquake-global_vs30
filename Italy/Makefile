#
# Makefile for inserting Italy Vs30 grid into the global mosaic.
#
# This is, perhaps just behind Greece, is a good Makefile to look at to understand 
# the workflow for regions with coastlines. This one is a bit more complicated than
# Greece because it "fixes" (similar to the lakes in the Iran map) two problem spots. 
# One spot is Venice, where there is a discrepancy between grdlandmask and the way 
# the authors created the map (explained below), and the other is a small, square 
# region in Sardinia where the authors did not fill in Vs30 values. 
#

include ~/.vs30/Constants.mk

IT_NORTH = 47.10833333333333
IT_SOUTH = 35.21666666666666
IT_WEST = 6.45833333333333
IT_EAST = 19.58333333333333

IT_BASE_REGION = $(IT_WEST)/$(IT_EAST)/$(IT_SOUTH)/$(IT_NORTH)

IT_NORTH_EXT = 48.1083333333333

IT_EXT_REGION = $(IT_WEST)/$(IT_EAST)/$(IT_SOUTH)/$(IT_NORTH_EXT)

BIG_EUR_REGION = 4/21/34/50

IT_VEN_REGION = 11.7/12.4/44/45

IT_ISLAND_REGION = 8.8/9.5/40.5/41

.PHONY: all plots clean clean_plots veryclean

all : new_italy.grd weights.grd

plots : plot_base_map plot_final_mask plot_weights plot_new_italy plot_raw_map \
        plot_it_non_zero_smooth plot_mask_a plot_new_mask plot_new_mask_mul plot_final_map_eur

clean : clean_plots
	$(RM) it_$(RES)c.grd north.grd final_mask.grd it_non_zero* *mask* \
	it_ext.grd weights.grd italy.grd gmt.history \
	it_ven_mask.grd new_italy.grd new_ven.grd

clean_plots :
	$(RM) *.ps *.png
	$(RM) eur_vs30.grd

veryclean : clean

######################################################################################
# weights.grd is plotted and explained: We have values that range from 0 to 1. 
# Areas outside the border will have values < 0.5, and inside > 0.5. 
# So we subtract 0.5 (making outside < 0, inside ranging 
# from 0 to 0.5), multiply by 2 (making outside still < 0,
# inside ranging from 0 to 1.0), and then keep only positive values (making
# outside = 0, inside ranging from 0 to 1).
#

weights.grd : final_mask.grd
	gmt grdmath -fg $< 0.5 SUB 2 MUL DUP 0 GT MUL = $@

######################################################################################
# final_mask.grd is plotted.
#

final_mask.grd : new_mask_mul_landmask_add_a.grd
	gmt grdmath $< DUP 0 GE MUL = $@

#
# new_mask_mul_landmask_add_a.grd is plotted.
#

new_mask_mul_landmask_add_a.grd : mask_a.grd new_mask_mul_landmask.grd
	gmt grdmath mask_a.grd new_mask_mul_landmask.grd ADD 1 SUB = $@

new_mask_mul_landmask.grd : new_mask.grd landmask.grd
	gmt grdmath new_mask.grd landmask.grd MUL = $@

#
# new_mask.grd is plotted.
#

new_mask.grd : it_non_zero.grd landmask_smooth.grd
	gmt grdmath it_non_zero.grd DUP NOT landmask_smooth.grd MUL ADD = $@

landmask_smooth.grd : landmask.grd ../src/smooth
	../src/smooth infile=landmask.grd fx=$(REGION_FX) fy=$(REGION_FY) outfile=$@

#
# mask_a.grd is plotted.
#

mask_a.grd : clipmask_smooth.grd landmask.grd
	gmt grdmath clipmask_smooth.grd landmask.grd MUL = $@

#
# These steps are complicated to describe, so I've indicated above at which "checkpoint"
# a plot is made. Note that the final goal is to have a mask which smooths the border between Italy
# and other countries when those borders touch, but not to smooth any of Italy's coastline
# or islands into the water.
# 
# The first plot of this portion of the workflow is made here, of "clipmask_smooth.grd"
#

clipmask_smooth.grd : clipmask.grd ../src/smooth
	../src/smooth infile=clipmask.grd fx=$(REGION_FX) fy=$(REGION_FY) outfile=$@

clipmask.grd : landmask_water.grd it_non_zero.grd
	gmt grdmath landmask_water.grd it_non_zero.grd ADD 0 GT = $@

it_non_zero.grd : new_italy.grd
	gmt grdmath $< 603 NEQ = $@

#####################################################################################################
# Create a mask over the small square of missing/603 nodes on the island west of Italy.
# Replace any values inside this small masked region which are GT 600 with 1000.
# Insert back into the larger Italy grid. This grid can now be inserted into the global mosaic.
#

new_italy.grd : island_landmask.grd new_ven.grd
	gmt grdmath island_landmask.grd NOT new_ven.grd MUL island_landmask.grd ADD = $@

island_landmask.grd : it_island_mask.grd landmask.grd new_ven.grd
	gmt grdmath  landmask.grd it_island_mask.grd EQ new_ven.grd MUL DUP 600 GT 1000 MUL EXCH DUP 600 LT MUL ADD = $@

it_island_mask.grd : it_island_coords.xy
	gmt grdmask it_island_coords.xy -I$(RES)s -R$(IT_EXT_REGION) -N0/0/1 -fg -G$@

######################################################################################################
# Insert the fixed ven region back into the larger Italy grid.
#

new_ven.grd : ven_landmask.grd it_ext.grd
	gmt grdmath ven_landmask.grd NOT it_ext.grd MUL ven_landmask.grd ADD = $@

#
# Replace all 603 nodes not in the lake and within the small "ven" region with 600. 
#

ven_landmask.grd : it_ven_mask.grd landmask.grd it_ext.grd
	gmt grdmath landmask.grd it_ven_mask.grd EQ it_ext.grd MUL DUP 603 EQ 3 MUL SUB = $@

#
# Two regions of this grid need to be "fixed" before the grid can be inserted. 
#
# The first, "ven", is the region just south of Venice in which the value 603 (unique
# to water and dry area outside Italy) has been assigned to dry land inside Italy. 
# The second is the small square on the island off the west coast of Italy, which has
# also been given the unique value of 603 despite being dry land inside Italy.
#
# To begin fixing these regions, first we make a mask over the "ven" region.
#

it_ven_mask.grd : it_ven_coords.xy
	gmt grdmask it_ven_coords.xy -I$(RES)s -R$(IT_EXT_REGION) -N0/0/1 -fg -G$@

###########################################################################################
# Create two landmasks - one which has value 1 for land and 0 for water, and one that
# 1 for water and 0 for land.
#

landmask_water.grd :
	gmt grdlandmask -V -R$(IT_EXT_REGION) -I$(RES)s -G$@ -Df -N1/0/1/0/0

landmask.grd :
	gmt grdlandmask -V -R$(IT_EXT_REGION) -I$(RES)s -G$@ -Df -N0/1/0/1/0

##############################################################################################
# We're going to paste a 1 degree chunk of nodes on top of the grid to help
# with smoothing later. Value for each node will be 603, consistent with the
# original grid.
#

it_ext.grd : it_$(RES)c.grd north.grd
	gmt grdpaste it_$(RES)c.grd north.grd -Git_ext.grd

north.grd :
	gmt grdmath -R$(IT_WEST)/$(IT_EAST)/$(IT_NORTH)/$(IT_NORTH_EXT) -I$(RES)s/$(RES)s 603 = $@

###########################################################################################
# Start by rescaling to 30-second resolution and shift the map to make it co-register
# with the global grid.
#

it_$(RES)c.grd : italygeol60s_vsgrid.grd
	gmt grdsample -nl+t0.1 -I$(RES)s/$(RES)s -fg -R$(IT_BASE_REGION) $< -G$@

italygeol60s_vsgrid.grd :
	echo "Italy grid file italygeol60s_vsgrid.grd must be supplied."

../src/smooth :
	$(MAKE) -C ../src smooth

#################################
#
# Plots
#

plot_base_map : base.png

plot_final_mask : final_mask.png

plot_weights : weights.png

plot_raw_map : italy_raw.png

plot_new_italy : new_italy.png

plot_it_non_zero_smooth : clipmask_smooth.png

plot_mask_a : mask_a.png

plot_new_mask : new_mask.png

plot_new_mask_mul : new_mask_mul_landmask_add_a.png

plot_final_map_eur : eur.png

# Set some flags for plotting. To change things for individual plots,
# make sure to only change it in the code block below.

Jflags = M12		# Mercator projection, 12 cm across
Bflags = a2f1WSen	# Tick marks. Every 2 degrees deneoted with number, 1 degree tickmarks, print numbers on W and S axes
Dflags = 14/4.3/9/0.5	# Scalebar - position 14, 4.3. 9 cm long, 0.5 cm wide.
Eflags = 720		# 720 dpi resolution for psconvert
Tflags = g		# Output png. Use f for pdf, j for jpg

# Plot the final Vs30 Map:

final_mask.png : final_mask.grd
	gmt grdimage $< -J$(Jflags) -C$(NEW_WEIGHTS_CPT) -B$(Bflags) -K > final_mask.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> final_mask.ps
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> final_mask.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) final_mask.ps

base.png : it_$(RES)c.grd
	gmt grdimage it_$(RES)c.grd -J$(Jflags) -C$(NEW_VS30_CPT) -B$(Bflags) -K > base.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_VS30_CPT) -O -K >> base.ps
	gmt pscoast -J$(Jflags) -R$(IT_BASE_REGION) -Df -O -N1 -N2 -W >> base.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) base.ps

new_italy.png : new_italy.grd
	gmt grdimage new_italy.grd -J$(Jflags) -C$(NEW_VS30_CPT) -B$(Bflags) -K > new_italy.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_VS30_CPT) -O -K >> new_italy.ps
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> new_italy.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) new_italy.ps

# Plot the clipping mask:
 
mask_a.png : mask_a.grd
	gmt grdimage mask_a.grd -J$(Jflags) -C$(NEW_WEIGHTS_CPT) -B$(Bflags) -K > mask_a.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> mask_a.ps
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> mask_a.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) mask_a.ps

new_mask.png : new_mask.grd
	gmt grdimage new_mask.grd -J$(Jflags) -C$(NEW_WEIGHTS_CPT) -B$(Bflags) -K > new_mask.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> new_mask.ps
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> new_mask.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) new_mask.ps

new_mask_mul_landmask_add_a.png : new_mask_mul_landmask_add_a.grd
	gmt grdimage new_mask_mul_landmask_add_a.grd -J$(Jflags) -C$(NEW_WEIGHTS_CPT) -B$(Bflags) -K > new_mask_mul_landmask_add_a.ps 
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> new_mask_mul_landmask_add_a.ps 
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> new_mask_mul_landmask_add_a.ps 
	gmt psconvert -E$(Eflags) -P -T$(Tflags) new_mask_mul_landmask_add_a.ps
 
# Plot the smoothed clipping mask:
 
clipmask_smooth.png : clipmask_smooth.grd
	gmt grdimage clipmask_smooth.grd -J$(Jflags) -C$(NEW_WEIGHTS_CPT) -B$(Bflags) -K > clipmask_smooth.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> clipmask_smooth.ps
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> clipmask_smooth.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) clipmask_smooth.ps
 
# Plot the weights:
 
weights.png : weights.grd
	gmt grdimage weights.grd -C$(NEW_WEIGHTS_CPT) -J$(Jflags) -B$(Bflags) -K > weights.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_WEIGHTS_CPT) -O -K >> weights.ps     
	gmt pscoast -J$(Jflags) -R$(IT_EXT_REGION) -Df -O -N1 -N2 -W >> weights.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) weights.ps
 
# Plot the raw Vs30 map:

italy_raw.png : italygeol60s_vsgrid.grd
	gmt grdimage italygeol60s_vsgrid.grd -J$(Jflags) -C$(NEW_VS30_CPT) -B$(Bflags) -K > italy_raw.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_VS30_CPT) -O -K >> italy_raw.ps
	gmt pscoast -R$(IT_BASE_REGION) -J$(Jflags) -N1 -N2 -W -Df -O >> italy_raw.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) italy_raw.ps

# Plot the new Vs30 map:

eur.png : eur_vs30.grd
	gmt grdimage eur_vs30.grd -J$(Jflags) -C$(NEW_VS30_CPT) -B$(Bflags) -K > eur.ps
	gmt psscale -D$(Dflags) -L -C$(NEW_VS30_CPT) -O -K >> eur.ps
	gmt pscoast -J$(Jflags) -R$(BIG_EUR_REGION) -O -Df -N1 -N2 -W >> eur.ps
	gmt psconvert -E$(Eflags) -P -T$(Tflags) eur.ps
	rm gmt.history

eur_vs30.grd : ../global_vs30.grd
	gmt grdcut -R$(BIG_EUR_REGION) $< -G$@
