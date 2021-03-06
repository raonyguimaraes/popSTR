#PopSTR - A Population based microsatellite genotyper

##Downloading and Installing popSTR

1.Download zip of source code: https://github.com/DecodeGenetics/popSTR/archive/master.zip

2.Unzip source code: `unzip popSTR-master.zip`

3.Move to source code directory: `cd popSTR-master/`

4.Unzip library dependencies: `unzip -qq seqan-library-1.4.1.zip` 
                              
                             unzip -qq liblinear-2.01.zip

5.Run make command: `make`

##Running popSTR

PopSTR has three steps and one must iterate between steps 2 and 3 five times or use the kernel provided(see bottom of file): 

###1. computeReadAttributes - Find useable reads, estimate number of repeats in them and compute their attributes for logistic regression.

Is run per individual, per chromosome.

Call:

    computeReadAttributes IN.bam outputDirectory markerInfoFile minFlankLength maxRepeatLength PN-id

Parameters:

* `IN.bam` - bam file containing reads from the individual specified in PN-id(the last parameter). An index (.bai file) for the bam file must also exist in the same directory. 
* `outputDirectory` - Directory to write output to, a folder called attributes will be created in the directory if it doesn't exist. A folder with the name of the chromosome being run will be created in the attributes folder if it doesn't exist. The output will be written to a file located at outputDirectory/attributes/chr/PN-id
* `markerInfoFile` - A file listing the markers to be genotyped, must only contain markers from one chromosome. Format for markerInfoFile:

        chrom startCoordinate endCoordinate repeatMotif numOfRepeatsInRef 1000refBasesBeforeStart 1000refBasesAfterEnd repeatSeqFromRef

* `minFlankLength` - Minimum number of flanking bases required on each side of a repeat for a read to be considered useful.
* `maxRepeatLength` - All alleles with a basePair length above this number will be lumped together into a greater than allele. Should be set close to 0.5 * readLength in IN.bam.
* `PN-id` - The id of the individual being genotyped.

##Output:
The resulting attribute-file has the following format:

The first line contains the PN-id and is followed by one or more pairs of a Markerline and one or more Attributelines

Markerline (one per each marker):

chrom startCoordinate endCoordinate repeatMotif numOfRepeatsInRef numOfReadsFound repeatSeqFromRef  A1-initialization A2-initialization

Attributelines (Number of lines is determined by numOfReadsFound from the Markerline):

numOfRepeatsInRead  alignmentQualBefore alignmentQualAfter  locationShift mateEditDist  repeatPurity  ratioOver20inRepeat ratioOver20AfterRepeat  sequenceLength  wasUnaligned  repeatSeqFromRead 

* numOfRepeatsInRead - The allele reported by the read, tells us how many repeats were found in the read.
* alignmentQualBefore - Quality of realignment on left side of repeat. Ranges from 0 to 1 and higher values indicate more reliable reads
* alignmentQualAfter - Quality of realignment on right side of repeat. Ranges from 0 to 1 and higher values indicate more reliable reads
* locationShift - Measures changes from original alignment during the realignment of flanking sequences, common values lie between 0 and 4.
* mateEditDist - Edit distance of aligned base pairs of the mate sequence to the reference, higher values of this attribute are suspicious.
* repeatPurity - Measures how well the repeat sequence from the read matches the repeat sequence from the reference. Ranges from 0 to 1 and higher values indicate more reliable reads
* ratioOver20inRepeat - Measures the ratio of bases in the repeat sequence with high quality scores. Ranges from 0 to 1 and higher values indicate more reliable reads
* ratioOver20inRepeat - Measures the ratio of bases in the right flanking sequence with high quality scores. Ranges from 0 to 1 and higher values indicate more reliable reads
* sequenceLength - Total length of read sequence, reads shorter than what is normal indicate less reliable reads.
* wasUnaligned - Boolean value indicating if the read was unaligned, unaligned reads are rarely processed and should be checked if causing conflict.


###2. computePnSlippage - Estimate individual specific slippage rates

Is run per individual.

Call:

    computePnSlippage attributesDirectory PN-id outputFile iterationNumber minPnsPerMarker markerSlippageDirectory modelAndLabelDir previousSlippageRate
    
Parameters:

* `attributesDirectory` - Should be the same as the "outputDirectory" parameter in the computeReadAttributes call. The program will go through the directory, find the attributes folder, in there it will find the folder for each chromosome and within it the list of useful reads and their attributes for the individual being processed.
* `PN-id` - The id of the individual being genotyped.
* `outputFile` - Slippage rate will be written to a file with the iterationNumber appended to this name, i.e. in iteration 0 outputFile0 will be created. This parameter should be the same for all PNs because the PN-id and slipppage rate will be appended to the file.
* `iterationNumber` - Zero based number of iteration.
* `minPnsPerMarker` - Minimum number of individuals used to estimate slippage at a given marker so the slippage can be considered reliable. Slippage rates estimated using too few individuals might not be accurate. 
* `markerSlippageDirectory` - ONLY applicable if iterationNumber > 0!
                               Should be a path to the marker slippage directory supplied in the msGenotyper call.
                               The program will go through the directory, find the folder for each chromosome and within it the markerSlippage file created in iteration
                               number [iterationNumber] of msGenotyper.
* `modelAndLabelDir` - ONLY applicable if iterationNumber > 0!
                        Should be a path to the logistic regression model and current genotype labels directory supplied in the msGenotyper call.
                        The program will go through the directory, find the folder for each chromosome and within it the labels file created in iteration number [iterationNumber]
                        of msGenotyper. It will also locate a file for each marker containing the current logistic regression model for that marker.
* `previousSlippageRate` - ONLY applicable if iterationNumber > 0!
                        The slippage rate estimated for this PN in iteration number [iterationNumber -1] (can be found by: grep <PN-id> [outputFile][iterationNumber-1] | cut -f 2)

###3. msGenotyper - Determine genotypes, estimate marker slippage rates and train logistic regression models.

Is run per chromosome but for all individuals at once.

Call:

    msGenotyper attDir PN-slippageFile startCoordinate endCoordinate intervalIndex markerSlippageDir modelAndLabelDir iterationNumber chromName vcfOutputDirectory vcfFileName

Parameters:

* `attDir` - attDir should be the outputDirectory supplied in the computeReadAttributes call and chromNum the chromosome being genotyped.
* `PN-slippageFile` - Should be the outputFile parameter supplied in the computePnSlippage call. Should contain one line per individual to be genotyped.
* `startCoordinate`/`endCoordinate` - If the user wishes to split the genotyping by intervals for paralellization purposes the start and end coordinates should be specified here. 
                      To turn this functionality off specify startCoordinate as 0 and endCoordinate>length(chromosome being genotyped)
* `intervalIndex` - If splitting the chromosome this parameter indicates the index of the interval specified. 
                    When this functionality is not desired the user can specify any number, it will be appended to names of all output files as "_intervalIndex" and must be manually
                    removed before performing subsequent iterations.
* `markerSlippageDir` - The marker slippage rates estimated will be written to [markerSlippageDir]/[chromName]/markerSlippage[iterationNumber]_[intervalIndex] and if iterationNumber>1 the marker slippage rates
                         estimated in the previous iteration will be read from [markerSlippageDir]/[chromName]/markerSlippageFile[iterationNumber-1]
* `modelAndLabelDir` -  In this directory a file called [PN-id]labels[iterationNumber]_[intervalIndex] will be created for each individual being genotyped and for each marker 
                                  a file called [chromNum]_[starCoordinateOfMarker] containing a logistic regression model for that marker will be created.
                                  If iteration number>1 the labels and models from the previous iteration will be read.
* `iterationNumber` - One based number of iteration.
* `chromName` - Name of chromosome to process.
* `vcfOutputDirectory` - ONLY applicable if iteration number == 5!
                          Directory to place vcf-files containing genotypes for all individuals at all markers contained within the interval being considered.
* `vcfFileName` - ONLY applicable if iteration number == 5!
                  Genotypes will be written to a vcf-file vcfOutputDirectory/vcfFileName_[intervalIndex].vcf
                  
####Additional notes:

When finishing an iteration of msGenotyper, two things must be done before starting the next iteration of computePnSlippage.

1. MarkerSlippage-files for all intervals must be concatenated into a single file for that chromosome.
   For example if one has just finished iteration 1 and the chromosome has been split into 20 intervals, the following command must be run in the directory containing the interval-split
   markerSlippage-files:
   
   `for i in {1..22}; do cat markerSlippageFile1_${i} >> markerSlippageFile1; done`
   
2. All label files for each individual must be concatenated into a single label file.
   Continuing the example from above the following command must be run where we assume the file pn-list contains a list of all PN-ids.
   
   `while read -r PN-id; do for i in {1..20}; do cat ${PN-id}labels1_${i} >> ${PN-id}labels1; done; done`
   
A specific vcf file will be created for each interval and when all intervals have finished running these must be manually concatenated into a single file.

    `grep ^# [vcfOutputDirectory]/vcfFileName_1.vcf > [vcfOutputDirectory]/vcfFileName.vcf`
    `for i in {1..[numIntervals]}; do grep -v ^# [vcfOutputDirectory]/vcfFileName_${i}.vcf >> [vcfOutputDirectory]/vcfFileName.vcf; done`
    
####Kernelization

To circumvent the iterative part using your entire dataset it is possible to train only slippage rates for a selected set of markers (a kernel) and run the "Default" versions of `computePnSlippage` and `msGenotyper` to obtain genotypes directly.
A kernel containing 8303 markers on chr1(HG38 coordinates) is supplied along with the software (kernelSlippageRates, kernelModels(1 file for each marker in the kernel) and kernelMarkersInfo).
To use the kernel one must follow the same three steps as before with a slight change:

#####1. Run computeReadAttributes using kernelMarkersInfo as parameter number 3 (markerInfoFile) for all PNs and write output to a separate directory. 

#####2. Run computeReadAttributes as described in the iterative version for all chromosomes on all PNs to be genotyped.

#####3. Run computePnSlippageDefault for all PNs (this version has an argument parser so the call is different).

Call:

    `computePnSlippageDefault -AF pathToOutputFile/from/computeReadAttributes/PN-id -OF outputFile -MS kernelSlippageRates -MD pathToKernelModelsFiles`

Parameters:

* `pathToOutputFile/from/computeReadAttributes/PN-id` - Output file from running computeReadAttributes with kernelMarkersInfo from the kernel as parameter number 3.
* `outputFile` - Slippage rate will be written/appended to this file. This parameter should be the same for all PNs because the PN-id and slipppage rate will be appended to the file.
* `kernelSlippageRates` - A file containing slippage rates and other info for markers in the kernel.
* `pathToKernelModelsFiles` - Logistic regression models for all markers in the kernel should be stored at this location.

#####4. Run msGenotyperDefault (this version has an argument parser so the call is different).

Call:

    `msGenotyperDefault -ADCN attributesDirectory/chromNum/ -PNS pnSlippageFile -MS markerSlippageFile -VD vcfOutputDirectory -VN vcfFileName -R start end -I intervalIndex`

Parameters:

* `attributesDirectory/chromNum/` - Should be a path to output files from running computeReadAttributes in the standard way followed by the name of the chromosome to be genotyped. This directory will be searched for attribute files for all PNs mentioned in the pnSlippage file.
* `pnSlippageFile` - This file should contain the output of computePnSlippageDefault for all individuals to be genotyped.
* `markerSlippageFile` - The slippage rates for all markers to be genotyped will be written to this file.
* `vcfOutputDirectory` - Directory to place vcf-files containing genotypes for all individuals at all markers contained within the interval being considered.
* `vcfFileName` - Genotypes will be written to a vcf-file vcfOutputDirectory/vcfFileName_[intervalIndex].vcf
* `start end` - If the user wishes to split the genotyping by intervals for paralellization purposes the start and end coordinates should be specified here. 
                      To turn this functionality off specify startCoordinate as 0 and endCoordinate>length(chromosome being genotyped)
* `intervalIndex` - If splitting the chromosome this parameter indicates the index of the interval specified. 
                    When this functionality is not desired the user can specify any number, it will be appended to names of all output files as "_intervalIndex" and must be manually
                    removed before performing subsequent iterations.
