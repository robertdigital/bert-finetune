# Fine-tuning BERT

**[BERT](https://arxiv.org/abs/1810.04805)** (Bidirectional Encoder Representations from Transformers) is a state-of-the-art NLP language model. BERT achieves state-of-the-art performance on a wide range of NLP tasks without any task specific architecture. Hence, we can fine-tune BERT or even combine it with other layers to create novel architectures. Since BERT is a large model, we require scaled-up training to iterate swiftly and enable productive experimentation.

In this repository, we provide a performance-optimized examples for training BERT at a single-node, multi-gpu scale. We explain how we deliver more than 3x performance improvement (over a baseline implementation) on a DGX-1V with V100 on BERT finetuning tasks for both BERTBASE and BERTLARGE.

![header](images/bert_header.jpg)

The code in this repository demonstrates:

* Using a [TFHub](https://www.tensorflow.org/hub) BERT module (see [list of BERT modules](https://tfhub.dev/s?q=bert)) wrapped in a Keras layer as a easy way to use BERT in a Keras model. We use Keras to keep the code simple and readable.
* [Performance Enhancements](#performance-enhancements). Applying the XLA compiler and Automatic Mixed Precision (AMP) to speed up BERT training significantly on NVIDIA Tensor Core GPUs.
* [Efficient multi-GPU training](#efficient-scaling-on-multiple-gpus). This is achieved using Horovod. We use FP16 compression and sparse-to-dense tensor conversion during communication to improve scaling performance.

## BERT Specifics

### Input Pipeline

BERT relies on specific preprocessing for the input sequences. This is described in their [paper](https://arxiv.org/abs/1810.04805). For the examples in this repository, we will use the preprocessing functions that have been adapted from the official GitHub repository ([google-research/bert](https://github.com/google-research/bert)). They are documented and placed in [bert_utils.py](https://github.com/NVAITC/examples/blob/master/bert_finetune/bert_utils.py).

The basic steps in the data input pipeline are:

1. Read the dataset into Python lists
2. Segment the dataset among worker processes
3. Create BERT tokenizer with `create_tokenizer_from_hub_module`
4. Convert Python lists into `InputExample` with `convert_text_to_examples`
5. Convert `InputExample` back into arrays with `convert_examples_to_features`
6. Feed arrays into model during training

You can view a Jupyter notebook that breaks down the process [here](bert_input.ipynb).

### BERT Keras Layer

The BERT layer ([see docstring](https://github.com/NVAITC/examples/blob/master/bert_finetune/bert_utils.py#L10)) can be instantiated easily:

```python
l = bert_utils.BERT(fine_tune_layers=TUNE_LAYERS, # number of cells to tune, "-1" means everything
                    bert_path=BERT_PATH,          # URL of TFHub BERT module
                    return_sequence=False,        # return sequence output for token-level tasks
                    output_size=H_SIZE,           # output/size of hidden representation
                    debug=False)                  # print debug output
```

Here is a simple notebook to demonstrate the instantiation of the of BERT layer: [link](https://github.com/NVAITC/examples/blob/master/bert_finetune/bert_layer.ipynb). Turning on the `debug` flag will cause a printout of the number of layers, as well as the dimensions and parameter count of all trainable variables in the instantiated layer.

Note that there are two main variants of the BERT model described in the paper and available on TFHub. 

| Model     | Hidden layers (L) | Hidden unit size (H) | Attention heads (A) | Feedforward filter size | Max sequence length | Parameters |
| --------- | ------------- | ---------------- | --------------- | ----------------------- | ------------------- | ---------- |
| BERTBASE  | 12 encoder    | 768              | 12              | 4 x  768                | 512                 | 110M       |
| BERTLARGE | 24 encoder    | 1024             | 16              | 4 x 1024                | 512                 | 330M       |

## Performance Enhancements

### Enabling Performance Features

By enabling performance features such as the [XLA](https://www.tensorflow.org/xla/overview) compiler and [Automatic Mixed Precision (AMP)](https://developer.nvidia.com/automatic-mixed-precision), we can get a performance improvement of about 3x on a DGX-1V system when training BERT.

<p align="center">
  <img src="https://raw.githubusercontent.com/NVAITC/bert-finetune/master/images/bertbase_perf.jpg" width="48%">
  <img src="https://raw.githubusercontent.com/NVAITC/bert-finetune/master/images/bertlarge_perf.jpg" width="48%">
</p>

Note that these benchmarks are run production DGX-1 on Max-Q (power efficiency) and hence are not formal guarantees of performance (your numbers will likely be about 20-30% higher). 

Create a TensorFlow session with AMP and XLA enabled:

```python
config = tf.ConfigProto()
# enable XLA
config.graph_options.optimizer_options.global_jit_level = tf.OptimizerOptions.ON_1
# enable AMP
config.graph_options.rewrite_options.auto_mixed_precision = True
sess = tf.Session(config=config)
tf.keras.backend.set_session(sess)

# In TF 1.x:
# Enable Resource Variables to improve efficiency of XLA fusions
tf.enable_resource_variables() 
```

Create the AMP optimizer and use it to compile the Keras model. The AMP optimizer performs dynamic loss scaling to avoid NaN/Inf and gradient underflow problems.

```python
opt = tf.train.experimental.enable_mixed_precision_graph_rewrite(opt)

model.compile(loss="sparse_categorical_crossentropy",
              optimizer=opt,
              metrics=["accuracy"])
```

### Efficient Scaling on Multiple GPUs

BERT has a large number of parameters (110M or 330M), which makes it important to reduce the communication overhead when synchronizing between workers by using a library such as [**Horovod**](https://github.com/horovod/horovod). Horovod uses NCCL (NVIDIA Collective Communications Library) which provides optimized implementation of inter-GPU communication operations, which can leverage the high-performance NVLINK or NVSWITCH interconnect between GPUs. We can get extremely good scaling efficiency (96%) when using Horovod.

<p align="center">
  <img src="https://raw.githubusercontent.com/NVAITC/bert-finetune/master/images/bert_scaling.jpg" width="80%">
</p>

>If you are new to Horovod or distributed training, please view:
>* [Basic Horovod MPI concepts](https://github.com/horovod/horovod/blob/master/docs/concepts.rst)
>* [Horovod Simple MNIST example](https://github.com/horovod/horovod/blob/master/examples/keras_mnist.py)
>* [NCCL](https://developer.nvidia.com/nccl)

BERT has a large Embedding layer which produces gradients as sparse [`IndexedSlices`](https://www.tensorflow.org/api_docs/python/tf/IndexedSlices) objects. As a result gradients are accumulated via allgather instead of allreduce. We use sparse to dense tensor conversion to force the gradients to be accumulated using allreduce instead, at the cost of increase memory usage. By futher using FP16 compression, we can increase the overall training performance of BERTBASE by about 40% as measured on a DGX-1V (8 V100 with NVLINK), or about 15% for BERTLARGE. 

```python
optimizer = hvd_keras.DistributedOptimizer(optimizer,
                                           compression=hvd.Compression.fp16,
                                           sparse_as_dense=True)
```

![BERT Horovod tuning](images/bert_horovod.jpg)

The performance increase for BERTLARGE is much less since the Embedding layer is less than half as large (9% in BERTLARGE vs 21% in BERTBASE).

<p align="center">
  <img src="https://raw.githubusercontent.com/NVAITC/bert-finetune/master/images/bertbase_params.jpg" width="48%">
  <img src="https://raw.githubusercontent.com/NVAITC/bert-finetune/master/images/bertlarge_params.jpg" width="48%">
</p>


**Impact of NVLINK Interconnect (DGX-1V)**

When we disable NVLINK during model training (`-x NCCL_P2P_DISABLE=1`), communication takes place exclusively over the PCIE bus and training performance drops from 25% (BERTBASE) to 50% (BERTLARGE) for the otherwise fully optimized training routine (FP16 compression, `sparse_as_dense=True`, XLA+AMP). 

### Additional Optimizations

**LazyAdam optimizer**

We use the LazyAdam optimizer to reduce the VRAM usage of the optimizer. It is a variant of the Adam optimizer that handles sparse gradient updates more efficiently. However, it provides slightly different semantics than the original Adam algorithm, and may lead to different empirical results (in practice there is no real model performance penalty)

This allows us to increase the batch size that we can use for training, measurably speeding up the training. For example, on a 16GB V100, the largest usable batch size is increased from 19 (Adam) to 25 (LazyAdam), resulting in 7% increase in training speed. We do not observe measurable improvements for BERTLARGE due to the proportionally smaller amount of sparse gradient updates.

  
## Example Task

### AG News Category Classification

This example demonstrates fine-grained news topic classification with 4 topics. There is a training set of 120000 articles and a test set of 7600 articles.

Each input example to be classified consists of the full headline + the article:

Run `python3 news_classification.py -h` to see the arguments that you can pass to the training script.

One liner for running in container for remote batch job:

```bash
# example to run on 8 GPUs (e.g. DGX-1)
bash -c 'export N_GPU=8 && git clone --depth 1 https://github.com/nvaitc/examples && cd examples/bert_finetune && bash train_news_classification_bertbase.sh'
```

#### Classification Performance

| Model      | Params | Test Acc |
| ---------- | ------ | -------- |
| XLNet      | 330M   | 0.955    |
| BERTLARGE  | 330M   | 0.946    |
| BERTBASE   | 110M   | 0.940    |
| CNN (Johnson, 2017) | ? | 0.934    |
| CNN (Zhang, 2015)   | ? | 0.905    |

To get better classification performance, we use learning rate warmup followed by square-root decay.

Note: As of TF 1.14, learning rate scheduling requires our [custom patch of TensorFlow](https://github.com/NVAITC/tensorflow-patched/releases/tag/v1.14.1-patch0). If you're using our container [`nvaitc/ai-lab:19.07`](https://cloud.docker.com/u/nvaitc/repository/docker/nvaitc/ai-lab) or newer, the patch is already included.

## Acknowledgements

**Core Maintainer**

Timothy Liu / [@tlkh](https://github.com/tlkh)

**References**

* BERT Authors: https://github.com/google-research/bert
* NVIDIA: https://github.com/NVIDIA/DeepLearningExamples/tree/master/TensorFlow/LanguageModeling/BERT
* TensorFlow Transformer Official Implementation: https://github.com/tensorflow/models/tree/master/official/transformer
