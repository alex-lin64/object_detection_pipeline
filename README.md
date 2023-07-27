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



cvat server username: django
pass: bfc


-------------------------------------------------------------------------------------------------------------
Work on instructions for use


Installation:

    To work with vdeos on fiftyone, installing ffmpeg is necessary and can be downloaded [here](https://ffmpeg.org/download.html).




FIFTYONE/CVAT Workflow:

    **NOTE** 
    As of now, images/videos used in the baseline are stored locally.  This means the links to the images are tied to each sample in Fiftyone are the location of the image on your local machine.  For consistency, use the data/ directory in baseline_system/ to store all iamges/videos required for use to avoid issues related to bad links.

    **NOTE**
    Datasets in Fiftyone can only consist of a single media type, either images, text, or video.  As such, when inferencing on a directory of videos or images, please ensure the directory contains only one type of media.





FAQ:

    Problem: 
        Conda env not allowing downsloads through Pip
    Solution: 
        Run the following command after activating the problematic conda env
            python -m pip install --upgrade --force-reinstall pip

