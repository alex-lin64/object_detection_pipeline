Tasks:
    -- cvat integration
    -- data storage
    -- connect from separate machine
    -- organize code/documentation
    -- make requirements file
    -- client for detectron2 


    Nice to haves:
        ** batching the inputs if input is directory?  how to?


docker run --gpus all --rm --ipc=host --shm-size=1g --ulimit memlock=-1 --ulimit stack=67108864 -p8000:8000 -p8001:8001 -p8002:8002 --mount type=bind,source="/mnt/c/Users/Alex Lin/Desktop/baseline_system/triton_deploy/models",destination=/models nvcr.io/nvidia/tritonserver:22.06-py3 tritonserver --model-repository=/models --strict-model-config=false --log-verbose 1

CVAT = localhost:8080
Fiftyone = localhost:5151
Triton = localhost:8001


cvat server username: django
pass: arclight


PUT NOTES IN FOR DEVELOPMENT ONLY FILES



-------------------------------------------------------------------------------------------------------------
## Welcome to the ARCLIGHT Baseline system!  

Below you will find installation instrucitons and an overview of the typical workflow through the baseline pipeline.  A FAQ section at the bottom tracks common errors and bugs that may arise during the setup phase.


## Installation:

<details><summary> <b>Expand</b> </summary>

``` shell 

# clone the repository locally
git clone git@git.bostonfusion.com:bfc/operations/multi-domain-systems/arclight/baseline_object_detection.git

# go to project folder
cd /baseline_object_detection

# install required packages in conda environment w/ python=3.8
pip install -r requirements.txt

# install ffmpeg (needed for fiftyone to work with videos), download from following link
https://ffmpeg.org/download.html

```

</details>

## Integrating Custom Models

[!IMPORTANT]
CUDA >= 11.7 is required to run the Triton Inference Server

Additional models added to the Triton Inference Server needs to be converted into the TensorRT engine format.  ONNX framework serves as an intermediary between pytorch and TensorRT enginer.  A workflow would be to export pytorch/tensorflow models in ONNX format, then export that through a docker container of the triton inference server as the TensorRT engine.

Example.

```bash
# 1. Export pytorch/tensorflow model as ONNX, usually a script in most model clones.  Please note that many conversion scripts to onnx require onnx-graphsurgeon, which is only compatible on linux.

# 2. ONNX -> TensorRT with trtexec and docker
docker run -it --rm --gpus=all nvcr.io/nvidia/tensorrt:22.06-py3

# 3. Copy onnx -> container: docker cp yolov7.onnx <container-id>:/workspace/

# 4. Export with FP16 precision, min batch 1, opt batch 8 and max batch 8
./tensorrt/bin/trtexec --onnx=yolov7.onnx --minShapes=images:1x3x640x640 --optShapes=images:8x3x640x640 --maxShapes=images:8x3x640x640 --fp16 --workspace=4096 --saveEngine=yolov7-fp16-1x8x8.engine --timingCacheFile=timing.cache

# 5. Test engine
./tensorrt/bin/trtexec --loadEngine=yolov7-fp16-1x8x8.engine

# 6. Copy engine -> host: docker cp <container-id>:/workspace/yolov7-fp16-1x8x8.engine .

#7. The baseline gitlab contains the /triton_deploy directory prepopulated with the detectron2 and yolov7 tensorrt engines.  Adding new tensorrt engines must adhere to the repository structure

$ tree triton-deploy/
triton-deploy/
└── models
    └── yolov7
        ├── 1
        │   └── model.plan
        └── config.pbtxt
    └── your_model
        ├── 1
        │   └── model.plan
        └── config.pbtxt

# Create folder structure 
mkdir -p triton-deploy/models/your_model/1/  # 1 indicates version #
touch triton-deploy/models/your_model/config.pbtxt
# Place model
mv your_model.engine triton-deploy/models/your_model/1/model.plan

# steps 2-7 credit to yolov7 deploy readme
```

Once the model repository is extended with the new model, populate the config file.  The array of input/s and output/s must **EXACTLY** match the variable name and dims that the model was exported with.  These input and output names and dims can by uploading the model's onnx file to [Netron](https://netron.app/).  Additional configurations for tensorrt engines are documented [here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html).

``` json
name: "your_model"
platform: "tensorrt_plan"
max_batch_size: 1
input [
  {
    name: "input_name"
    data_type: input_type
    dims: [input dims]
  }
  ...
]
output [
  {
    name: "output_names"
    data_type: output_type
    dims: [output dims]
  },
  ...
]
```

## Initialization: Triton Inference Server

``` shell
docker run --gpus all --rm --ipc=host --shm-size=1g --ulimit memlock=-1 --ulimit stack=67108864 -p8000:8000 -p8001:8001 -p8002:8002 --mount type=bind,source="path/to/triton_deploy/models",destination=/models nvcr.io/nvidia/tritonserver:22.06-py3 tritonserver --model-repository=/models --strict-model-config=false --log-verbose 1

```

In the log you should see:

```
+------------+---------+--------+
| Model      | Version | Status |
+------------+---------+--------+
| yolov7     | 1       | READY  |
+------------+---------+--------+
| detectron2 | 1       | READY  |
+------------+---------+--------+
| ...        | 1       | READY  |
+------------+---------+--------+
```

The inference server will be reachable via port 8001.


## Initialization: CVAT

_optional (but not right now). CVAT is locally (until the ARCLIGHT server is up to host the application)

Follow the official CVAT [installation guide](https://opencv.github.io/cvat/docs/administration/basics/installation/).  

After installation, start CVAT with 

```shell

cd /path/to/cvat/

# to start
CVAT_VERSION=v2.3.0 docker compose up -d

# to stop (has to be within cvat directory)
docker compose down
```

CVAT can be accessed locally at localhost:8080


## Workflow

[!IMPORTANT]
Datasets in Fiftyone can only consist of a single media type, either images, text, or video.  As such, when inferencing on a directory of videos or images, please ensure the directory contains only one type of media.

[!NOTE]
As of now, images/videos used in the baseline are stored locally.  This means the links to the images are tied to each sample in Fiftyone are the location of the image on your local machine.  For consistency, use the data/ directory in baseline_system/ to store all iamges/videos required for use to avoid issues related to bad links.  **Ideally, all data should be stored in a dedicated directory on the ARCLIGHT server**.






    





FAQ:

    Q: Triton Inference Server starts, but the models are not ready
    A: This is usually a problem with the config files.  The input and output names and dims have to EXACTLY match the ones the model was serialized with.  Missing commas and closing brackets can also cause the error. 

    Q: Fiftyone stuck on error screen after deleting a dataset
    A: This is due to the way fiftyone handles deletes and exceptions.  Just restart fiftyone by rerunning the cell containing start_session().

    Q: Triton Inference Server displays 'defaulting to cpu' during inference
    A: The server is still using gpu, this can be verified through the Resource Monitor on windows or similar app on another OS.

    Q: Conda environments not allowing downloads through pip
    A: Run "python -m pip install --upgrade --force-reinstall pip" inside the conda env

