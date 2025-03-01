# Mitigating Covariate Shift with Style Transfer
Shuvadeep Saha, Martin Keenan

## Resources

- Style transfer code based on the [Neural Transfer Using Pytorch](https://pytorch.org/tutorials/advanced/neural_style_tutorial.html) tutorial by Alexis Jacq.
- YOLO object detection model based on [yolov5](https://github.com/ultralytics/yolov5) implemented by ultralytics.
- [Long-term Thermal Drift Dataset](https://www.kaggle.com/datasets/ivannikolov/longterm-thermal-drift-dataset) published by [Nikolov et al.](https://openreview.net/forum?id=LjjqegBNtPi).

## Project Description
In goal of this project is to experiment with methods for training object detection models that are robust to data drift. We use the Long-term Thermal Drift Dataset, which is dataset collected across five months from an outdoor thermal-imaging camera mounted on the outside of a building. The dataset includes 651 images annotated with bounding boxes on pedestrians.

Our hypothesis is that using neural style transfer to make older training images look more similar to images taken just before the evaluation period will improve our model's performance. In our experiment we consider three scenarios, shown below, in which we train a model to evaluate on March, April, and August data. (The dataset does not include data for May, June, and July.) In each scenario the training set consists of all images taken during previous months. This simulates a real-world situation where one would only have access to images captured since the camera was deployed.

<p align="center">
<img src=".github/scenarios.png" width=50% height=100% 
class="center">
</p>

For each scenario, we create a new "stylized" training set. Images from the last ten days of training data are designated as style images and the rest as content images. We apply style to each of the content images by matching each to the style image taken at the closest time-of-day, then apply the neural style transfer algorithm.
<p align="center">
<img src=".github/style_transfer_white.png" width=40% height=100% 
class="center">
</p>

The figure below shows an example of style transfer from Scenario 2. A style image from the end of March is matched to a content image from February. After style transfer, the new image replaces the February image in the stylized training set.
<p align="center">
<img src=".github/style_example.png" width=70% height=100% 
class="center">
</p>

To evaluate whether our methodology adds any value to the model, we then finetune two YOLOv5 pretrained object detection models. The first is trained on the original training set and serves as a baseline. The second is trained on the new stylized training set. We then compare the two models on the same test dataset that includes images from one month after the latest training image.
<p align="center">
<img src=".github/experiment_evaluation_white.png" width=50% height=100% 
class="center">
</p>

## Repository Description
The code can be divided into two stages: style transfer and object detection. 

The file [stylized_datasets.py](stylized_datasets.py) creates the baseline and style-augmented datasets used for training object detection models. The code for implementing the neural style transfer algorithm is in the [style](style) directory. Examples of the style-augmented datasets are in [datasets](datasets) and [datasets_highstyle](datasets_highstyle).

The directory [yolov5s](yolov5s) contains code for the YOLOv5 object detection model. The commands in [main.ipynb](main.ipynb) train the YOLOv5 model and save the results for the datasets present in [datastes](datasets) and [datasets_highstyle](datasets_highstyle).

## Instructions

### Style Transfer
To create the baseline and style-augmented training datasets and the test datasets, first create a new directory that contains a subdirectory with the annotated images from the [Long-term Thermal Drift Dataset](https://www.kaggle.com/datasets/ivannikolov/longterm-thermal-drift-dataset). Then run the following Python command using the path to the new directory.
```
python style_transfer.py --datadir <PATH_TO_DATASET_DIRECTORY> --content_weight 10
```
The `content_weight` option controls the degree of style added to content images by changing the weight of the content loss. For the low-style experiments we set `content_weight` equal to 10 and for the high-style experiments we set it equal to 1.

### Object Detection
To run object detection, first run the [main.ipynb](main.ipynb). The relative directory path of the `train.py` file must be specified and is present in `yolov5s/train.py`. For more information on the type of arguments that can be passed, one can refer to the file train.py. The next step is to specift the relative path for the config file which contains the train/val/test dataset directories and the number of classes we are dealing with. The data folder is located at `yolov5s/data`. All the yaml conig files must be stored in this directory. Below shows an example on train/val/test on a March unstylized images dataset.
```
!python yolov5s/train.py --img 288 --batch 128 --epochs 250 --data detect-drift-March-un.yaml --weights yolov5s.pt --cache
```
Similarly for validation, specify the relative path to the saved weights and the same data file path as used during training.
```
!python yolov5s/val.py --weights yolov5s/runs/train/detect-drift-March-un-train/weights/best.pt \
--data yolov5s/data/detect-drift-March-un.yaml --img 288
```
Lastly for testing the model on any custom image, video, etc, specify the path of the image/video/etc file for detection:
```
!python yolov5s/detect.py --weights yolov5s/runs/train/April/weights/best.pt \
--conf-thres 0.5 --line-thickness 2 --hide-conf --source test-vid-2.mp4
```

## Results
The results of all the runs, are stored in `yolov5s/runs`. The performance on the baseline (unstylized) images are shown below. The first set of three bars show the mean average precision on March, April, and August of a model trained on January and February images. The decrease in performance as the model is evaluated on images captured later and later in the year indicates the presence of drift. Also, as we include more images that are closer in date to the test images, the performance of the model increases. This is shown by the increasing orange and green bars.
<p align="center">
<img src=".github/performance on the baseline.png" width=50% height=100% 
class="center">
</p>

The performance of the stylized images and the comparison with the baseline is shown below. The difference between the model trained on the stylized dataset and baseline dataset was zero when test on March images, positive on April images, and negative on August images. This indicates there is evidence our approach can be useful but warrants further investigation on other datasets.
<p align="center">
<img src=".github/Performance on the stylized images.png" width=50% height=100% 
class="center">
</p>

## References

- Jacq, Alexis. “Neural Transfer Using Pytorch.” https://pytorch.org/tutorials/advanced/neural_style_tutorial.html.

- Glenn Jocher, et al., Ultralytics, yolov5: v6.2 - YOLOv5 Classification Models, Reproducibility, ClearML and Deci.ai integrations (v6.2), (2022)
- Nikolov, Ivan Adriyanov, et al. "Seasons in Drift: A Long-Term Thermal Imaging Dataset for Studying Concept Drift." Thirty-fifth Conference on Neural Information Processing Systems, (2021).
- Zheng, Xu, et al. "Stada: Style transfer as data augmentation." arXiv preprint arXiv:1909.01056, (2019).
