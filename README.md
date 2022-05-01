# Object Detection in an Urban Environment

## Prerequisites

### Local docker setup

For local setup if you have your own Nvidia GPU, you can use the provided Dockerfile and requirements in the [build directory](./build).

Follow [the README therein](./build/README.md) to create a docker container and install all prerequisites.

## Data

For this project, we will be using data from the [Waymo Open dataset](https://waymo.com/open/).

### Download the Waymo dataset

The first goal of this project is to download the data from the Waymo's Google Cloud bucket to your local machine. For this project, we only need a subset of the data provided (for example, we do not need to use the Lidar data). Therefore, we are going to download and trim immediately each file. In `download_process.py`, you can view the `create_tf_example` function, which will perform this processing. This function takes the components of a Waymo Tf record and saves them in the Tf Object Detection api format. An example of such function is described [here](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html#create-tensorflow-records). We are already providing the `label_map.pbtxt` file.

```
cd src/data_processing
python3 download_process.py --data_dir <folder_to_save_dataset> --size <number of files you want to download>
```

An example command is `python3 download_process.py --data_dir /app/project/data --size 100`

The time taken to process 100 files is around 3 hours.

### Exploratory Data Analysis

This is the most important task of any machine learning project. After the data is downloaded to the folder specified in the previous step, we will explore and visualise a sample of the dataset to have a sense of what the data looks like.

To do so, open the `Exploratory Data Analysis` notebook and run the cells to get some visualisation charts and figures. An example is shown below.

![Initial data visualisation](/assets/initial_data_exploratory.png "Initial data visualisation")

From the exploratory dataset, we can immediately note down 2 things. First, there are many more cars compared to pedestrains and cyclists. Pedestrains and cyclists will form our subset of "rare classes" that we need to pay closer attention to. Second, the lighting and weather conditions vary across datasets. We need to ensure that there is proper representation of images with good and bad lighting, as well as good and adverse weather conditions.

For additional exploratory data analysis, we will check the spread of classes across each image to see how heavily the number of cars skew the dataset. We will also need to see check the distribution of lighting conditions. For that, we will find the average pixel value and covert to perceived brightness for each image.

#### Class distribution

![Initial data visualisation](/assets/cls_distribution.png "Initial data visualisation")

#### Brightness distribution

![Initial data visualisation](/assets/brightness_distribution.png "Initial data visualisation")

From the data above, there are many more cars than other classes in the dataset, and the brightness values are skewed towards the brighter area. In the data augmentation step, these issues need to be addressed.

### Split the dataset

The dataset is split randomly in the ratio 80% for training, 10% for validation and 10% for testing.

The three subfolders are:

* /app/project/data/processed/train

* /app/project/data/processed/val

* /app/project/data/processed/test

```
cd src/data_processing
python3 create_splits.py --source <source_folder_of_tfrecords> --destination <destination_folder_of_tfrecords>
```

An example command is `python3 create_splits.py --source /app/project/data/processed --destination /app/project/data/processed`

## Model

### Download base model

The config that we will use for this project is `pipeline.config`, which is the config for a SSD Resnet 50 640x640 model. You can learn more about the Single Shot Detector [here](https://arxiv.org/pdf/1512.02325.pdf).

First, let's download the [pretrained model](http://download.tensorflow.org/models/object_detection/tf2/20200711/ssd_resnet50_v1_fpn_640x640_coco17_tpu-8.tar.gz).

```
chmod +x download_model.sh
cd scripts
./download_model.sh /app/project/src/experiments/pretrained_model
```

### Experiments folder structure
The experiments folder will be organized as follow:

```
experiments/
    - experiment_0/ - create a new folder for each experiment you run
    - experiment_1/ - create a new folder for each experiment you run
    - pretrained_model/
    - reference - reference training with the unchanged config file
    - edit_config.py - edit pipeline.config
    - exporter_main_v2.py - to create an inference model
    - label_map.pbtxt - label to class mapping file
    - model_main_tf2.py - to launch training
```

### Creating a new config file for training

We need to edit the config files to change the location of the training and validation files, as well as the location of the label_map file, pretrained weights. We also need to adjust the batch size. To do so, run the following:

```
cd experiments
python3 edit_config.py \
--train_dir /app/project/data/processed/train \
--eval_dir /app/project/data/processed/val \
--batch_size 2 \
--checkpoint /app/project/src/experiments/pretrained_model/ssd_resnet50_v1_fpn_640x640_coco17_tpu-8/checkpoint/ckpt-0 \
--label_map /app/project/src/experiments/label_map.pbtxt
```

A new config file `pipeline_new.config` will be created, which will be put in a newly created `experiment_N` folder, where N is the experiment number.

## Training

### Start training

After the new config file is put to a new experiments folder, the training can begin with the below command.

```
cd src/experiments
python3 model_main_tf2.py --model_dir=<path_to_experiment_folder>/ --pipeline_config_path=<path_to_experiment_folder>/pipeline_new.config

```

Example command: `python3 model_main_tf2.py --model_dir=experiment_0/ --pipeline_config_path=experiment_0/pipeline_new.config`

### Viewing Tensorboard

We can visualise the loss and learning rate trends by using Tensorboard.

```
tensorboard --logdir <path_to_experiment_folder>
```

Example command: `tensorboard --logdir experiment_0`

### Evaluation

Once the training is finished, launch the evaluation process:
*
 an evaluation process:

```
python3 experiments/model_main_tf2.py --model_dir=experiments/reference/ --pipeline_config_path=experiments/reference/pipeline_new.config --checkpoint_dir=experiments/reference/
```





### Improve the performances

Most likely, this initial experiment did not yield optimal results. However, you can make multiple changes to the config file to improve this model. One obvious change consists in improving the data augmentation strategy. The [`preprocessor.proto`](https://github.com/tensorflow/models/blob/master/research/object_detection/protos/preprocessor.proto) file contains the different data augmentation method available in the Tf Object Detection API. To help you visualize these augmentations, we are providing a notebook: `Explore augmentations.ipynb`. Using this notebook, try different data augmentation combinations and select the one you think is optimal for our dataset. Justify your choices in the writeup.

Keep in mind that the following are also available:
* experiment with the optimizer: type of optimizer, learning rate, scheduler etc
* experiment with the architecture. The Tf Object Detection API [model zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/tf2_detection_zoo.md) offers many architectures. Keep in mind that the `pipeline.config` file is unique for each architecture and you will have to edit it.

**Important:** If you are working on the workspace, your storage is limited. You may to delete the checkpoints files after each experiment. You should however keep the `tf.events` files located in the `train` and `eval` folder of your experiments. You can also keep the `saved_model` folder to create your videos.


### Creating an animation
#### Export the trained model
Modify the arguments of the following function to adjust it to your models:

```
python experiments/exporter_main_v2.py --input_type image_tensor --pipeline_config_path experiments/reference/pipeline_new.config --trained_checkpoint_dir experiments/reference/ --output_directory experiments/reference/exported/
```

This should create a new folder `experiments/reference/exported/saved_model`. You can read more about the Tensorflow SavedModel format [here](https://www.tensorflow.org/guide/saved_model).

Finally, you can create a video of your model's inferences for any tf record file. To do so, run the following command (modify it to your files):
```
python inference_video.py --labelmap_path label_map.pbtxt --model_path experiments/reference/exported/saved_model --tf_record_path /data/waymo/testing/segment-12200383401366682847_2552_140_2572_140_with_camera_labels.tfrecord --config_path experiments/reference/pipeline_new.config --output_path animation.gif
```

## Submission Template

### Project overview
This section should contain a brief description of the project and what we are trying to achieve. Why is object detection such an important component of self driving car systems?

### Set up
This section should contain a brief description of the steps to follow to run the code for this repository.

### Dataset
#### Dataset analysis
This section should contain a quantitative and qualitative description of the dataset. It should include images, charts and other visualizations.
#### Cross validation
This section should detail the cross validation strategy and justify your approach.

### Training
#### Reference experiment
This section should detail the results of the reference experiment. It should includes training metrics and a detailed explanation of the algorithm's performances.

#### Improve on the reference
This section should highlight the different strategies you adopted to improve your model. It should contain relevant figures and details of your findings.
