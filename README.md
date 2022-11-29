Author: [https://umerjamil16.github.io/](https://umerjamil16.github.io/)

# Containerizing Kaldi - the speech recognition toolkit

Kaldi is a very popular Automatic Speech Recognition (ASR) toolkit. In my experience, setting up Kaldi on a system can take huge amount of time. Therefore, it is best to use containerization approach.

Following is the Dockerfile that I wrote. We will be using a cuda enabled Ubuntu 18.04 image from Nvidia NGC:

```bash
FROM nvidia/cuda:11.1.1-cudnn8-devel-ubuntu18.04

LABEL Author="umerjamil16@github.io" Email="umer1694@gmail.com"
LABEL Description="Kaldi ASR Image" Vendor="x" Version="1.0"

RUN sh -c 'echo "APT { Get { AllowUnauthenticated \"1\"; }; };" > /etc/apt/apt.conf.d/99allow_unauth'
RUN apt -o Acquire::AllowInsecureRepositories=true -o Acquire::AllowDowngradeToInsecureRepositories=true update
RUN apt-get install -y curl wget
RUN apt-key del 7fa2af80
RUN wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-keyring_1.0-1_all.deb
RUN dpkg -i cuda-keyring_1.0-1_all.deb
RUN rm -f /etc/apt/sources.list.d/cuda.list /etc/apt/apt.conf.d/99allow_unauth cuda-keyring_1.0-1_all.deb
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys A4B469963BF863CC F60F4B3D7FA2AF80
RUN apt-get update && apt-get upgrade -y
RUN apt-get install git -y

WORKDIR /opt
RUN git clone  --depth 1 https://github.com/kaldi-asr/kaldi.git

WORKDIR /opt/kaldi/tools
RUN apt-get install zlib1g-dev automake autoconf unzip wget git sox gfortran libtool subversion python2.7 python3 flac gawk swig python-pip nano -y
RUN ./extras/install_mkl.sh 
RUN pip install numpy
#RUN pip  install git+https://github.com/sequitur-g2p/sequitur-g2p@master
RUN make -j  $(nproc) 
#RUN ./extras/install_irstlm.sh
RUN ./extras/install_srilm.sh name org email
RUN ./extras/install_sequitur.sh 
RUN chmod +x env.sh 
RUN ./env.sh

WORKDIR /opt/kaldi/src/
RUN  ./configure --shared --use-cuda
RUN  make depend -j  $(nproc)  
RUN  make -j  $(nproc) 
RUN echo "SUCCESS"
WORKDIR /opt/kaldi/
```
 
 ## Building the image
To build the docker image, ``cd`` to folder where Kaldi Dockerfile is located and run the following command.

```
docker build -t kaldi_asr:latest -f Dockerfile .
```
This will build the image with the ``kaldi_asr`` name.

## Running the kaldi container

First let us create a ``working_directory`` to mount it to our container. The ``working_dir`` will contain our Kaldi project files and will allow data persist. 

```
mkdir working_dir
docker run -it --rm -v ./working_dir:/working_dir --name kaldi_docker kaldi_asr:latest
```
To attach bash shell to the above running container,
```
docker exec -it kaldi_docker /bin/bash
```
Now you can run your kaldi scripts inside this container and work on your projects.

