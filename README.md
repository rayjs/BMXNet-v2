# BMXNet 2 // Hasso Plattner Institute

A fork of the deep learning framework [mxnet](http://mxnet.io) to study and implement quantization and binarization in neural networks.

This project is based on the [first version of BMXNet](https://github.com/hpi-xnor/BMXNet), but is different in that it reuses more of the original MXNet operators.
This aim was to have only minimal changes to C++ code to get better maintainability with future versions of mxnet.

## mxnet version

This version of BMXNet 2 is based on: *mxnet v1.5.1*

## News

See all BMXNet changes: [Changelog](CHANGELOG.md).

- **May 21, 2019**
    - Model converter for deployment released ([Test & Example](tests/binary/test_converter.py))
- **Sep 01, 2018**
    - We rebuilt BMXNet to utilize the new Gluon API for better maintainability
    - To build binary neural networks, you can use drop in replacements of convolution and dense layers (see [Usage](#usage)):
    - Note that this project is still in beta and changes might be frequent

# Setup

If you only want to test the basics, you can also look at our [docker setup](#docker-setup).

We use [CMake](https://cmake.org/download/) to build the project.
Make sure to [install all the dependencies described here](docs/install/build_from_source.md#prerequisites).
If you install CUDA 10, you will need CMake >=3.12.2

Adjust settings in cmake (build-type ``Release`` or ``Debug``, configure CUDA, OpenBLAS or Atlas, OpenCV, OpenMP etc.).

Further, we recommend [Ninja](https://ninja-build.org/) as a build system for faster builds (Ubuntu: `sudo apt-get install ninja-build`).

```bash
git clone --recursive https://github.com/hpi-xnor/BMXNet-v2.git # remember to include the --recursive
cd BMXNet-v2
mkdir build && cd build
cmake .. -G Ninja # if any error occurs, apply ccmake or cmake-gui to adjust the cmake config.
ccmake . # or GUI cmake
ninja
```

#### Build the MXNet Python binding

Step 1 Install prerequisites - python, setup-tools, python-pip and numpy.
```bash
sudo apt-get install -y python-dev python3-dev virtualenv
wget -nv https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py
python2 get-pip.py
```

Step 1b (Optional) Create or activate a [virtualenv](https://virtualenv.pypa.io/).

Step 2 Install the MXNet Python binding.
```bash
cd <mxnet-root>/python
pip install -e .
```

If your mxnet python binding still not works, you can add the location of the libray to your ``LD_LIBRARY_PATH`` as well as the mxnet python folder to your ``PYTHONPATH``:
```bash
$ export LD_LIBRARY_PATH=<mxnet-root>/build/Release
$ export PYTHONPATH=<mxnet-root>/python
```

## Training

Make sure that you have a new version of our example submodule [example/bmxnet-examples](https://github.com/hpi-xnor/BMXNet-v2-examples/):
```bash
cd example/bmxnet-examples
git checkout master
git pull
```

Examples for hyperparameters are documented in the [Wiki](https://github.com/hpi-xnor/BMXNet-v2-wiki/blob/master/hyperparameters.md).

## Inference

To speed up inference and compress your model, you need to save it as a symbol (not with gluon) and afterwards convert it with the model-converter.
Please check the corresponding [test case](tests/binary/test_converter.py).
```bash
build/tools/binary_converter/model-converter model-0000.params
```

## Tests

To run BMXNet specific tests install `pytest`:
```bash
pip install pytest
```

Then simply run:
```bash
pytest tests/binary
```

## Usage

We added binary versions of the following layers of the gluon API:
- gluon.nn.Dense -> gluon.nn.QDense
- gluon.nn.Conv1D -> gluon.nn.QConv1D
- gluon.nn.Conv2D -> gluon.nn.QConv2D
- gluon.nn.Conv3D -> gluon.nn.QConv3D

## Overview of Changes

We added three functions `det_sign` ([ada4ea1d](https://github.com/hpi-xnor/BMXNet-v2/commit/ada4ea1d4418cfdd6cbc6d0159e1a716cb01cd85)), `round_ste` ([044f81f0](https://github.com/hpi-xnor/BMXNet-v2/commit/044f81f028887b9842070df28b28de394bd07516)) and `contrib.gradcancel` to MXNet (see [src/operator/contrib/gradient_cancel[-inl.h|.cc|.cu]](src/operator/contrib)).

The rest of our code resides in the following folders/files:
- Examples are in a submodule in [example/bmxnet-examples](https://github.com/hpi-xnor/BMXNet-v2-examples)
- Tests are in [tests/binary](tests/binary)
- Layers are in [python/mxnet/gluon/nn/binary_layers.py](python/mxnet/gluon/nn/binary_layers.py)
- Converter is in [tools/binary_converter](tools/binary_converter)

For more details see the [Changelog](CHANGELOG.md).

## Docker setup

A docker image for testing of BMXNet can be build similar to our CI script at [.gitlab-ci.yml](.gitlab-ci.yml), however it only supports CPU, so actual training might be tedious.

```bash
cd ci
docker build -f docker/Dockerfile.build.ubuntu_cpu --build-arg USER_ID=1000 --build-arg GROUP_ID=1000 --cache-from bmxnet2-base/build.ubuntu_cpu -t bmxnet2-base/build.ubuntu_cpu docker
```

Then you can enter the container (and automatically delete it)
```bash
docker run --rm -it bmxnet2-base/build.ubuntu_cpu # deletes the container after running
docker run -it bmxnet2-base/build.ubuntu_cpu # keeps the container after running (it needs to be removed manually later)
```

Inside the container you can now clone, build and test BMXNet 2
```bash
# clone
mkdir -p /builds/
cd /builds/
git clone https://github.com/hpi-xnor/BMXNet-v2.git bmxnet --recursive
cd bmxnet
# build
mkdir build
cd build
cmake -DBINARY_WORD_TYPE=uint32 -DUSE_CUDA=OFF -DUSE_MKL_IF_AVAILABLE=OFF -GNinja ..
cd ..
cmake --build build
export PYTHONPATH=/builds/bmxnet/python # add python binding
# run the tests (we need to upgrade pytest first via pip3)
pip3 install pytest --upgrade
pytest tests/binary
```

You can even train a simple binary MNIST model, but you might need to update the examples to the newest version first (checkout the master branch).
```bash
cd example/bmxnet-examples/mnist/
git checkout master
pip3 install mxboard
python3 mnist-lenet.py --bits 1 # trains a binary lenet model with 1 bit activations and 1 bit weights on MNIST
```

### Citing BMXNet 2

Please cite [our paper](https://arxiv.org/abs/1812.01965) about BMXNet 2 in your publications if it helps your research work:

```text
@article{bmxnetv2,
  title = {Training Competitive Binary Neural Networks from Scratch},
  author = {Joseph Bethge and Marvin Bornstein and Adrian Loy and Haojin Yang and Christoph Meinel},
  journal = {ArXiv e-prints},
  archivePrefix = "arXiv",
  eprint = {1812.01965},
  Year = {2018}
}
```

### References

- [XNOR-Net: ImageNet Classification Using Binary Convolutional Neural Networks](https://arxiv.org/abs/1603.05279)
- [Binarized Neural Networks: Training Deep Neural Networks with Weights and Activations Constrained to +1 or -1](https://arxiv.org/abs/1602.02830)
- [DoReFa-Net: Training Low Bitwidth Convolutional Neural Networks with Low Bitwidth Gradients](https://arxiv.org/abs/1606.06160)
