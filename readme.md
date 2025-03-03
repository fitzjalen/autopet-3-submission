# [autoPET III Winning Solution] MICCAI 2024 Challenge Submission: Team LesionTracer

🏆 This repository provides the code for running our approach to the [AutoPET III Challenge](https://autopet-iii.grand-challenge.org/) (*Team LesionTracer*) which was awared the first place in the model centric category. When using this repository, please cite our paper:

**From FDG to PSMA: A Hitchhiker's Guide to Multitracer, Multicenter Lesion Segmentation in PET/CT Imaging** 

&nbsp; &nbsp;   [![arXiv](https://img.shields.io/badge/arXiv-2409.09478-b31b1b.svg)](http://arxiv.org/abs/2409.09478)


Authors:  
Maximilian Rokuss, Balint Kovacs, Yannick Kirchhoff, Shuhan Xiao, Constantin Ulrich, Klaus H. Maier-Hein and Fabian Isensee


Author Affiliations:  
Division of Medical Image Computing, German Cancer Research Center (DKFZ), Heidelberg  
Faculty of Mathematics and Computer Science, Heidelberg University

# Overview

Our model builds on [nnU-Net](https://github.com/MIC-DKFZ/nnUNet) with a [ResEncL architecture](https://github.com/MIC-DKFZ/nnUNet/blob/master/documentation/resenc_presets.md) preset as a strong baseline. We then introduce several improvements:

- We adjusted the plans file using a patch size of  ```[192,192,192]``` such that the model has a high contextual understanding. You can find the the plans [here](nnunetv2/architecture/nnUNetResEncUNetLPlansMultiTalent.json) (remember to change the "dataset_name" field to your dataset name).
- The model is pretrained on a diverse collection of medical imaging datasets to establish a strong anatomical understanding, which we then fine-tune on the autoPET III challange dataset. The pretrained checkpoint we use for *initializing the model before training* is availabe [here](https://zenodo.org/records/13753413) (Dataset619_nativemultistem). Note, that this is not the final model checkpoint.
- The model is trained using [misalignment data augmentation](https://github.com/MIC-DKFZ/misalignment_DA) as well as omitting the smoothing term in the dice loss calcuation.
- We use a dual-headed architecture for organ and lesion segmentation which improves performance as well as speeds up convergence, especially in cases without lesions.

**You can [download the final checkpoint here](https://zenodo.org/records/13786235)!**

## Getting started

### Installation

We recommend to create a new conda environment and then run:


```bash
git clone https://github.com/MIC-DKFZ/autopet-3-submission.git
cd autopet-3-submission
pip install -e .
```

### Preprocessing

Download the [autoPET dataset](https://autopet-iii.grand-challenge.org/dataset/) and use the standard nnUNet preprocessing pipeline. You can adjust the number of processes for faster processing. You can freely chose your dataset number which we quote as DATASET_ID_LESIONS.

```bash
nnUNetv2_plan_and_preprocess -d DATASET_ID_LESIONS -c 3d_fullres -np 20 -npfp 20
```

### Extract organs from CT image

In order to train on the organ segmentation as a secondary objective we use [TotalSegmentator](https://github.com/wasserth/TotalSegmentator) to predict 10 anatomical structures in part relevant to PET tracer uptake: spleen, kidneys, liver, urinary bladder, lungs, brain, heart, stomach, prostate, and glands in the head region (parotid glands, submandibular glands).

You can either:

1.    Use our predicted organ masks which we made availabe [in this repo](nnunetv2/preprocessing/organ_extraction/autopet3_organ_labels), or

2.    Follow these instructions for your own dataset: This one is a bit time-consuming to redo, so hang with me here. First, install TotalSegmentator using ```pip install TotalSegmentator```. We did it in a separate environment. Then put [this script](nnunetv2/preprocessing/organ_extraction/predict_and_extract_organs.py) into your nnUNet_raw dataset directory for autoPETIII. Running this script can take a long time for the whole dataset (several days on our machines) since the TotalSegmentator inference is not optimized to handle several cases simultaneously.

        ```bash
        python predict_and_extract_organs.py
        ```

When this step is done, copy the raw nnUNet dataset such that you have a new dataset which is identical. The original dataset containing lesions annotations should have a different DATASET_ID_ORGANS than the original DATASET_ID_LESIONS. E.g. "Dataset200_autoPET3_lesions" and "Dataset201_autoPET3_organs". Then exchange the content of the  ```labelsTr``` folder with the provided [organ labels](nnunetv2/preprocessing/organ_extraction/autopet3_organ_labels) or in case you ran the above script (TotalSegmentator inference) use the labels from ```labelsTr_organs```. Now run the preprocessing again for the new dataset. Important: do not use the ```--verify_dataset_integrity``` flag.

Lastly, to combine the datasets run

```bash
nnUNetv2_merge_lesion_and_organ_dataset -l DATASET_ID_LESIONS -o DATASET_ID_ORGANS
```

Now you are good to go to start a training. Use the dataset with DATASET_ID_LESIONS for any further steps. If everything runs smoothly you could discard the dataset folder with DATASET_ID_ORGANS.

> If you want to do the last step step manually head over to the preprocessed folder. You should have two folders now with the different preprocessed datasets, containing lesion or organ segmentations. Navigate to the dataset containing the organ labels and then into the folder ```nnUNetPlans_3d_fullres```. Either unpack the ```.npz``` files using a script or start a default nnUNet training on the organ dataset such that they are automatically unpacked to ```.npy``` files. Now we are almost there. Search for all files ending with ```_seg.npy``` and rename them to have the ending ```_seg_org.npy```. Finally copy these files into the ```nnUNetPlans_3d_fullres``` folder of the preprocessed dataset containing the lesion segmentations. That's it - easy right?


### Training

Training the model can be simply achieved by [downloading the pretrained checkpoint](https://zenodo.org/records/13753413) (Dataset619_nativemultistem) and running:

```bash
nnUNetv2_train DATASET_ID_LESIONS 3d_fullres 0 -tr autoPET3_Trainer -p nnUNetResEncUNetLPlansMultiTalent -pretrained_weights /path/to/pretrained/weights/fold_all/checkpoint_final.pth
```

We train a five fold cross-validation for our final submission.


### Inference

After training your own model or [downloading our final checkpoint here](https://zenodo.org/records/13786235) you can use the standard nnUNet inference, for more information see [here](https://github.com/MIC-DKFZ/nnUNet/blob/master/documentation/how_to_use_nnunet.md). MODEL_FOLDER refers to the folder containing all 5 folds. Remember that the files in the input folder have to be named according to the nnUNet format, i.e. the CT files end with *_0000.nii.gz* and the PET images with *_0001.nii.gz*.

```bash
nnUNetv2_predict_from_modelfolder -i INPUT_FOLDER -o OUTPUT_FOLDER -m MODEL_FOLDER
```


Happy coding! 🚀

# Citation


```
@article{rokuss2024fdgpsmahitchhikersguide,
      title={From FDG to PSMA: A Hitchhiker's Guide to Multitracer, Multicenter Lesion Segmentation in PET/CT Imaging}, 
      author={Maximilian Rokuss and Balint Kovacs and Yannick Kirchhoff and Shuhan Xiao and Constantin Ulrich and Klaus H. Maier-Hein and Fabian Isensee},
      journal={ArXiv},
      year={2024},
      publisher={arXiv},
      eprint={2409.09478},
      archivePrefix={arXiv},
      url={https://arxiv.org/abs/2409.09478}, 
}
```

# Acknowledgements
<img src="documentation/assets/HI_Logo.png" height="100px" />

<img src="documentation/assets/dkfz_logo.png" height="100px" />

nnU-Net is developed and maintained by the Applied Computer Vision Lab (ACVL) of [Helmholtz Imaging](http://helmholtz-imaging.de) 
and the [Division of Medical Image Computing](https://www.dkfz.de/en/mic/index.php) at the 
[German Cancer Research Center (DKFZ)](https://www.dkfz.de/en/index.html).