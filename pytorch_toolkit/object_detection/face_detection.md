# Real time face detector

This repository contains training scripts for lightweight SSD-based face detector. The detector is based on the MobileNetV2 backbone and has single SSD head with manually designed anchors. As a result it has computational complexity 0.51 GMACs and 1.03 M of parameters.


## Prerequisites

1. Download the [WIDER Face](http://shuoyang1213.me/WIDERFACE/) and unpack it to `data` folder.
2. Annotation in the VOC format can be found in this
[repo](https://github.com/sovrasov/wider-face-pascal-voc-annotations.git). Move the annotation files from
`WIDER_train_annotations` and `WIDER_val_annotations` folders to the `Annotation` folders inside the corresponding
directories `WIDER_train` and `WIDER_val`. Also annotation lists `val.txt` and `train.txt` should be copied to
`data/WIDERFace` from `WIDER_train_annotations` and `WIDER_val_annotations`.
The directory should be like this:

```
data
└── WIDERFace
    ├── WIDER_train
    │   ├──0--Parade
    │   ├── ...
    │   └── Annotations
    ├── WIDER_val
    │   ├──0--Parade
    │   ├── ...
    │   └── Annotations
    ├── val.txt
    └── train.txt
```

## Training

1. Download pre-trained MobileNetV2 weights `mobilenet_v2.pth.tar` from: [https://github.com/tonylins/pytorch-mobilenet-v2](https://github.com/tonylins/pytorch-mobilenet-v2). Move the file with weights to the folder `snapshots`.
   Or [checkpoint](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/wider_face_tiny_ssd_075x_epoch_70.pth) that was trained on Wider.  
2. To train the detector on a single GPU run in terminal:
  ```bash
  python3 ../../external/mmdetection/tools/train.py \
  configs/mobilenetv2_tiny_ssd300_wider_face.py
  ```


## Validation

1. To dumb detection of your model:
  ```bash
  python3 ../../external/mmdetection/tools/test.py    \
    configs/mobilenetv2_tiny_ssd300_wider_face.py   \
    <CHECKPOINT>   \
    --out result.pkl
  ```

2. Then run
  ```bash
  python3 ../../external/mmdetection/tools/voc_eval.py    \
    result.pkl    \
    configs/mobilenetv2_tiny_ssd300_wider_face.py
  ```
  One should observe 0.305 AP on validation set. For more detailed results and comparison with vanilla SSD300 see `../../external/mmdetection/configs/wider_face/README.md`.

## Conversion to OpenVINO format

1. Convert PyTorch model to ONNX format: run script in terminal
  ```bash
  python3 tools/onnx_export.py \
        configs/mobilenetv2_tiny_ssd300_wider_face.py
        <CHECKPOINT> \
        face_detector.onnx
  ```

2. Convert ONNX model to OpenVINO format with Model Optimizer: run in terminal
  ```bash
  mo.py --input_model face_detector.onnx \
    --scale 255 \
    --reverse_input_channels \
    --output_dir=./IR \
    --data_type=FP32
  ```
  This produces model `face_detector.xml` and weights `face_detector.bin` in single-precision floating-point format
  (FP32). The obtained model expects normalized image in planar BGR format.
  此处发现训练处的安全帽模型，在经上述命令转换后，无法识别，原因是转换模型的时候参数一定要根据需要变化，把上述命令中的--scale 255
  去掉后，检测正常，用的是openvino中自带的object_detection_sample_ssd

## Python demo

To run the demo connect a webcam end execute command:
```bash
python3 tools/detection_live_demo.py  \
  configs/mobilenetv2_tiny_ssd300_wider_face.py \
  <CHECKPOINT> \
  --cam_id 0
```


## Estimate theoretical computational complexity

To get per layer computational complexity estimations run the following command:
```bash
python3 tools/count_flops.py configs/mobilenetv2_tiny_ssd300_wider_face.py
```


## Fine-tuning

* Dataset should have the same data layout as WIDER Face in VOC format
 described in this instruction.
* Fine-tuning steps are the same as step 2 for training, but some adjustments in config are needed:
  - specify initial checkpoint containing a valid detector in `load_from` field of config
    `configs/mobilenetv2_tiny_ssd300_wider_face.py`
  - edit `data` section of config to pass a custom dataset.
