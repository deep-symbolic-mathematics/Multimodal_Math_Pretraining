# `SNIP`: A Multimodal Symbolic-Numeric Pretraining for Math (MathCLIP)

Official Implementation of **ICLR 2024 Spotlight** paper **[SNIP: Bridging Mathematical Symbolic and Numeric Realms with Unified Pre-training](https://openreview.net/forum?id=KZSEgJGPxu)**. 

[Paper](https://openreview.net/forum?id=KZSEgJGPxu) | [Models](https://drive.google.com/drive/folders/1oGVQPAuTwWQnhX_pxN3OdKDt9-rmCfV3?usp=share_link) | [Data](https://github.com/deep-symbolic-mathematics/Multimodal-Math-Pretraining/blob/main/run_export_data.sh) | [Results](https://github.com/deep-symbolic-mathematics/Multimodal-Symbolic-Regression/tree/main/srbench_results)

## Overview
Inspired by the great performance of [CLIP](https://arxiv.org/abs/2310.02227) in vision-language representation learning, we introduce a multi-modal pre-training model for symbolic mathematics, known as **SNIP** for **Symbolic-Numeric Integrated Pre-training**, which emphasizes the significance of numeric-augmented representations in math representation learning. 

<p align="center">
<img src="./images/SNIP3.gif" width="100%" /> 
 <br>
<b>SNIP: A multi-modal transformer model that connects symbolic math equations with numeric data representations using contrastive learning</b>
</p>



## Installation

The code requires dependencies specified in `environment.yml`. Please follow the relevant libraries to install or run:

```
conda env create -f environment.yml
```
This library requires `python>3.7`



## Pretrained Models
We've released two pretrained SNIP models, each designed for different types of analysis. Download them [here](https://drive.google.com/drive/folders/1oGVQPAuTwWQnhX_pxN3OdKDt9-rmCfV3?usp=share_link). You'll find:

- **[SNIP-10dmax](https://drive.google.com/drive/u/1/folders/1oGVQPAuTwWQnhX_pxN3OdKDt9-rmCfV3):** This model handles **up to 10-dimensional inputs**. More info in Section 5 and Appendix D p.3 of [paper](https://arxiv.org/pdf/2310.02227.pdf).

- **[SNIP-1d-normalized](https://drive.google.com/drive/u/1/folders/1oGVQPAuTwWQnhX_pxN3OdKDt9-rmCfV3):** This model is for **1-dimensional inputs** with **normalized targets**, great for focusing on function patterns. More details in Section 4 and Appendix D of [paper](https://arxiv.org/pdf/2310.02227.pdf).

To use them, create a `weights/` folder in your project, download the checkpoints there, and use the `--reload_model` parameter with the model path, like `--reload_model ./weights/snip-1d-normalized.pth`."


## Pretraining Data Generation
For pretraining, we generate synthetic data of (symbolic, numeric) pairs for math functions, based on method from [SymbolicMathematics](https://openreview.net/forum?id=S1eZYeHFDS). Each pair includes data points $(x,y)$ and a math function $f$ such that $y=f(x)$. See `generate_datapoints` function [here](https://github.com/deep-symbolic-mathematics/Multimodal-Math-Pretraining/tree/main/snip/envs/generators.py) for more info. You can also adjust data generation settings [here](https://github.com/deep-symbolic-mathematics/Multimodal-Math-Pretraining/tree/main/snip/envs/environment.py). 

The data is generated on-the-fly during training, but if you want to create and analyze it beforehand, use `run_export_data.sh`:
```
python train.py --export_data True --dump_path ./dump --max_input_dimension 10
```
Your exported data will be saved in the `data.prefix` file.


## SNIP Pretraining
All training settings for SNIP are in `parsers.py`. SNIP uses Transformer encoders for both symbolic and numeric heads, which you can find in the `encoder_f` and `encoder_y` modules [here](./snip/model/__init__.py). For information on contrastive learning and training, look at the [trainer](./snip/trainer.py) file. Here's how you can start the training:
```
python train.py --loss_type CLIP \
                --batch_size 256 \
                --dump_path ./dump \
                --max_input_dimension 10 \
                --exp_id run1-10d \
                --lr 4e-5 \
                --latent_dim 512 \
                --save_periodic 10
```
Feel free to adjust training and data settings in `parsers.py` and `environment.py` under `snip/envs/`. After running the command, the model trained for every 10 (`save_periodic`) epochs is saved in `dump/` path.


## Using SNIP for Cross-modal Property Prediction
Here we have provided code to test SNIP representations for the cross-modal symbolic-to-numeric property prediction tasks, meaning that in these tasks, the input is the symbolic mathematical equation and the label is the propery defined based on numeric data observations. 

### Data Generation 
To try it out, start by generating data. For instance, to generate 10k training examples for the **Non-Convexity Ratio (NCR)** prediction task (as explained in [paper](https://arxiv.org/pdf/2310.02227.pdf)), use this command:
```
python train.py --export_data True --is_proppred True --property_type ncr --dump_path ./dump --max_input_dimension 1 --n_steps_per_epoch 625  --exp_name data --exp_id ncr
```

This saves data for `ncr` property in `dump/data/ncr/`. To generate data for other properties, just change the `--property_type` parameter. 

### Training 
For this task, we use a Transformer encoder architecture (to encode symbolic equation inputs), followed by a regression predictor head (to predict the property). Training is done using Mean Squared Error (MSE) loss. Following are the commands for training different model variants defined in Sec 4 of [paper](https://arxiv.org/pdf/2310.02227.pdf). 

**Supervised Model (without Pretrining)**:
```
python train.py --is_proppred True \
                --property_type ncr \
                --reload_data functions,dump/data/ncr/train.prefix,dump/data/ncr/train.prefix, \
                --normalize_y True \
                --batch_size 64 \
                --dump_path ./dump \
                --max_input_dimension 1 \
                --exp_name NCR_pred \
                --exp_id run1 \
                --lr 1e-5 \
                --latent_dim 512 \
                --save_periodic 10
```

**SNIP Encoder (frozen)**:
```
python train.py --reload_model ./weights/snip-1d-normalized.pth --freeze_encoder True [other parameters] 
```

**SNIP Encoder (finetune)**:
```
python train.py --reload_model ./weights/snip-1d-normalized.pth --freeze_encoder False [other parameters] 
```

With these commands, the model saves automatically every 10 epochs. To use SNIP's encoder, you should activate `--reload_model` parameter with the path of model weights. You can also freeze the encoder with `--freeze_encoder True`.


### Inference
To test how well your models perform for each property prediction task, use the `run_eval_proppred.sh` script. For example, if you want to test the NCR property task, use this command:
```
python eval_proppred.py --is_proppred True \
                        --property_type ncr \
                        --reload_model dump/NCR/model.pth \
                        --reload_data functions,dump/data/ncr/test.prefix,dump/data/ncr/test.prefix,
```
This command will use the `--reload_model` parameter to load the weights of your trained model and test it against the dataset specified in the `--reload_data` path.


## Using SNIP for Symbolic Regression
To use SNIP for more complex tasks such as Symbolic Regression (uncovering symbolic math equations from data: numeric-to-symbolic generation task), check **[Multimodal-Symbolic-Regression](https://github.com/deep-symbolic-mathematics/Multimodal-Symbolic-Regression)** repository.

### Final Results on SRBench 
Our experimental results of SNIP on SRBench datasets for symbolic regression are provided in the [srbench_results/](https://github.com/deep-symbolic-mathematics/Multimodal-Symbolic-Regression/tree/main/srbench_results) directory in the [Multimodal-Symbolic-Regression](https://github.com/deep-symbolic-mathematics/Multimodal-Symbolic-Regression) repository. These results are shared to help the research community reproduce our paper's findings and serve as reference benchmarks. Each result file contains detailed performance metrics and experimental configurations used in our evaluations.


## Citation
If you find the paper or the repo helpful, please cite it with
<pre>
@inproceedings{
meidani2024snip,
title={{SNIP}: Bridging Mathematical Symbolic and Numeric Realms with Unified Pre-training},
author={Kazem Meidani and Parshin Shojaee and Chandan K. Reddy and Amir Barati Farimani},
booktitle={The Twelfth International Conference on Learning Representations},
year={2024},
url={https://openreview.net/forum?id=KZSEgJGPxu}
}
</pre>


## License 
This repository is licensed under MIT licence.

This work is built on top of other open source projects, including [Deep Learning for Symbolic Mathematics](https://github.com/facebookresearch/SymbolicMathematics) and [Contrastive Language-Image Pretraining](https://github.com/openai/CLIP). We thank the original contributors of these works for open-sourcing their valuable source codes.


## Contact Us
For any questions or issues, you are welcome to open an issue in this repo, or contact us at mmeidani@andrew.cmu.edu, and parshinshojaee@vt.edu.
