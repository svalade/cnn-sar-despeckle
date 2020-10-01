# Learned SAR Speckle Filter

## Principal components

### Pipeline

Please note that all code was written for and tested in Python3.

Code dependencies are stored in `requirements_cpu.txt` and `requirements_gpu.txt`, depending on the type of processor that will be used.
These can be installed in a virtual environment:

```
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
pip3 install -r requirements_cpu.txt
```

Then commands can be run from within the virtual environment such as

```
python3 filter.py \
  images/sample.tif \
  -w weights/lin.hdf5 \
  --single_channel_output \
  -o images/filtered_sample.tif
```

An overview of this research can be found on the [project website](https://tim-dav.github.io/cnn-sar-despeckle/).

-----------------------

**filter.py**

This is the complete pipeline for processing raw SAR inputs into despeckled
intensity images using a pretrained model.

It depends on `tif2intensity.py` for converting the raw SAR input into a
(speckled) intensity image, `predictor.py` for predicting despeckled image
patches, and `model.util` for helper functions.

```
usage: filter.py [-h] -o OUTPUT
                 [--channels_last] -w WEIGHTS
                 [--logspace]
                 [--mean_correction MEAN_CORRECTION]
                 [--no_weighting] [-r]
                 [-s STRIDE]
                 [--single_channel_output]
                 input

positional arguments:
  input                 tif image or
                        directory of tif
                        images

optional arguments:
  -h, --help            show this help
                        message and exit
  -o OUTPUT, --output OUTPUT
                        output file or
                        directory
  --channels_last       indicate if the image
                        channels are stored
                        last
  -w WEIGHTS, --weights WEIGHTS
                        which weights to use
                        for processing
  --logspace            provided weights were
                        trained in logspace
  --mean_correction MEAN_CORRECTION
                        when to apply mean-
                        correction to images
                        [smart|always|never]
  --no_weighting        do not center-weight
                        image patches
  -r                    recursively process
                        subdirectories
  -s STRIDE, --stride STRIDE
                        stride when
                        processing images
                        (default 192)
  --single_channel_output
                        create a separate
                        file for each channel
                        (polarity)
```

-----------------------

**tif2intensity.py**

These are helper functions for turning a raw SAR image into a (speckled) intensity
image.

-----------------------

**predictor.py**

This is a class for making predictions using specified weights.


### Training

**Dataset generation**

The model was trained using 2-channel intensity images, with the first channel for
the VH intensity image and the second channel for the VV intensity image. These 2-channel
intensity images were created from 4-channel raw SAR images using `tif2intensity.py`.

The data must be batched into sets of identical views, for which `batcher.py` was used.
This takes a `.csv` file as input that indicates to which set each image belongs (*see*
example below) and batches each set into its own directory. Set number 0 is reserved for
"bad" images that are either in an incorrect format or do not fit into any other set.
These will be ignored by the batcher. Negative numbered sets are validation images
and will be excluded from the training data by `model.pairgenerator.py` during training,
but still copied to the output directory for later use in validation.

| volcano    | orbit    | image                     | set   |
|------------|---------:|---------------------------|------:|
|Ambrym      |        59| S1_20181211T182129_59_int |      0|
|...         |       ...| ...                       |    ...|



-----------------------

**train.py**

Actual training of the model is handled by `train.py`. It depends on `model.py` for
the model specification, `pairgenerator.py` for loading the dataset, and `util.py` for
helper functions

```
usage: train.py [-h] --image_dir IMAGE_DIR [--batch_size BATCH_SIZE]
                [--nb_epochs NB_EPOCHS] [--lr LR] [--steps STEPS]
                [--weight WEIGHT] [--output_path OUTPUT_PATH] [--model MODEL]
                [--min_date_separation MIN_DATE_SEPARATION]
                [--logspace LOGSPACE]

train noise2noise model

optional arguments:
  -h, --help            show this help message and exit
  --image_dir IMAGE_DIR
                        image dir for input and target values (default: None)
  --batch_size BATCH_SIZE
                        batch size (default: 32)
  --nb_epochs NB_EPOCHS
                        number of epochs (default: 20)
  --lr LR               learning rate (default: 0.01)
  --steps STEPS         steps per epoch (default: 32)
  --weight WEIGHT       weight file for restart (default: None)
  --output_path OUTPUT_PATH
                        checkpoint dir (default: checkpoints)
  --model MODEL         model architecture (default: red30)
  --min_date_separation MIN_DATE_SEPARATION
                        Minimum date between image pair acquisition (default:
                        6)
  --logspace LOGSPACE   Convert images to logspace before training (default:
                        False)
```

The following command was used for linear-space training:

```shell
python ./model/train.py --image_dir ./vhvv_sets/ --batch_size 16 --nb_epochs 1000 --lr 0.001 --steps 32 --output_path ./checkpoints_vhvv_lin/ --min_date_separation 90
```

The following command was used for logarithmic-space training:

```shell
python ./model/train.py --image_dir ./vhvv_sets/ --batch_size 16 --nb_epochs 1000 --lr 0.001 --steps 32 --output_path ./checkpoints_vhvv/ --min_date_separation 90 --logspace

```

**model.py**

This contains the specification for the architecture of the model.

**pairgenerator.py**

This contains the `imgloader` class, which extends `keras.utils.Sequence` and is used for
loading the dataset during training. All loading is done through on-the-fly loading
of randomly selected, matching image patches.

### Evaluation

Evaluation of the results was done using `compare_psnr` and `compare_ssim` from `skimage.measure`.
All images for two volcanoes (Ertaale and Pitonfournaise) were either dedicated to evaluation sets
or excluded from training set, so that no knowledge of these specific volcanoes was acquired by
the CNN during the training process.

Lee, Kuan, and Frost filters were taken from the [PyRadar](https://pypi.org/project/pyradar/) package.
SAR-BM3D filtering was conducted using v1.0 of the filter available from the [UNINA website](http://www.grip.unina.it/research/80-sar-despeckling/80-sar-bm3d.html).

