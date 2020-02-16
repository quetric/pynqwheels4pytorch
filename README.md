# PYNQ Wheels for PyTorch
This is a collection of wheels for PyTorch and Torchvision build on top of PYNQ images. As opposed to most other PyTorch wheels for ARM, ours provide support for `torch.distributed`

# Prerequisites
All wheels are built in a chroot environment on Ubuntu, using the PYNQ-provided Qemu static binary for aarch64. This flow has been tested on Ubuntu 16.04 and 18.04. Install host-side requirements with:
```
sudo apt-get install chroot unzip kpartx
```
Download and unzip PYNQ SD-card image and clone the PYNQ repo. The following tutorial assumes the image file is called `pynq.img` and the PYNQ repo was cloned in the same folder where the image file resides.

# Build Tutorial

## Set up Chroot
First we need to mount the PYNQ image to a folder:
```
mkdir img
./PYNQ/sdbuild/scripts/mount_image.sh pynq.img img/
```
You will be asked for your `sudo` password. After the script completes, `img/` will be populated with the contents of the PYNQ SDcard image. We now need to mount a few host-side directories to the image: a workspace folder for more space, and `/proc`, `/dev`, `/sys` for various functionality we'll need during the build.
```
mkdir workspace
sudo mount --bind `pwd`/workspace `pwd`/img/workspace
sudo mount -o bind /proc img/proc
sudo mount -o bind /dev img/dev
sudo mount -o bind /sys img/sys
```
To enable networking in the chroot, we need to copy the config to the image mount (make sure it's copied and not symlinked!).
```
sudo cat /etc/resolv.conf > img/etc/resolv.conf
```
Next we need to register qemu as the default interpreter for ARM (aarch64) binaries. You'll need to be root to do this.
```
echo ':qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64-static:' > /proc/sys/fs/binfmt_misc/register
```
Set-up completed! Now enter the chroot to the PYNQ image.
```
sudo chroot `pwd`/img /bin/bash
```

## Build PyTorch
First we need to install some prerequisites in the PYNQ image
```
apt-get -y update && apt-get -y install protobuf-compiler libopencv-dev libopenmpi-dev
pip3 install setuptools
```
Clone PyTorch and switch to the desired version.
```
cd /workspace
git clone --recursive https://github.com/pytorch/pytorch.git -b v1.1.0
```
Set up your environment to enable/disable components as desired. Note that PyTorch 1.5 fails during QNNPACK build due to a [bug](https://github.com/pytorch/pytorch/issues/33124).
```
export USE_CUDA=OFF
export USE_MKLDNN=OFF
export USE_NNPACK=OFF
export USE_QNNPACK=OFF
export USE_PYTHON_QNNPACK=OFF
export USE_OPENCV=1
export BUILD_CAFFE2_MOBILE=OFF
export BUILD_CAFFE2_OPS=OFF
export BUILD_CUSTOM_PROTOBUF=OFF
export BUILD_TEST=0
```
Now build the binary wheel:
```
cd pytorch
python3 setup.py bdist_wheel
```
Build times will vary depending on the configuration you set and the number of cores available. The resulting wheel is in `/workspace/pytorch/dist/`. Install it with:
```
pip3 install /workspace/pytorch/dist/*.whl
```

## Build Torchvision
Once PyTorch is installed, we can build Torchvision. Mind the PyTorch version installed when choosing a Torchvision version. The only other dependency is Pillow. Note that Pillow 7 is only supported in Torchvision 0.5 and above.
```
pip3 install "Pillow<7"
```
Clone the repo:
```
git clone https://github.com/pytorch/vision.git -b 0.4.0
```
And build the wheel:
```
cd vision
python3 setup.py bdist_wheel 
```

## Cleaning Up
Exit the chroot with `exit`, then unmount every host-side folder mounted to the image:
```
umount img/workspace
umount img/proc
umount img/dev
umount img/sys
```
Unmount the image itself using the PYNQ script:
```
./PYNQ/sdbuild/scripts/unmount_image.sh img/ pynq.img
```
The script will unmount the image file and loop partitions created for it. The `workspace/` folder contains your wheels.

# References
The build methodology is based on info gathered from tutorials [here](https://github.com/hypriot/qemu-register/blob/master/register.sh) and [here](https://gist.github.com/luk6xff/9f8d2520530a823944355e59343eadc1). Alternative PYNQ-compatible PyTorch wheel available [here](https://github.com/chunter18/PyTorch-AARCH64) (built with NO_DISTRIBUTED=1). Additional PyTorch and Torchvision wheels are available [here](https://github.com/nmilosev/pytorch-arm-builds) but are built against Python 3.7 and therefore incompatible with PYNQ. 
