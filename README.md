# Dask on Fargate Cluster

Dask is a powerful Python library to operate parallel or distributed computation. We can leverage distributed computation on powerful cluster like [Fargate](https://aws.amazon.com/fargate/). Dask can be used very easily with Scikit-learn or XGBoost to distribute and parallelize the training. It requires a bit of a setting, but once it is set, it works like a charm. 



The easiest way to run Dask on Fargate is through a fantastic library build around Dask, called [dask-cloudprovider](https://github.com/dask/dask-cloudprovider). This library will take care of every settings required on AWS, such as permission for user and ressources. To know more about the library, please read this [post](https://medium.com/rapids-ai/getting-started-with-rapids-on-aws-ecs-using-dask-cloud-provider-b1adfdbc9c6e) on Medium. 

There are 3 steps to build a cluster on Fargate in order to use Dask paired with Scikit learn or XGBoost:



1. Create a local Anaconda environment
2. Create A docker images
3. Push the Docker images on Docker Hub
4. Run the cluster on Jupyter

AWS *Fargate* is a compute engine for Amazon ECS and EKS that allows to run containers without having to manage servers or clusters. To be able to use [dask-cloudprovider](https://github.com/dask/dask-cloudprovider) on Fargate, we need to be sure all librairies we need are installed on the Docker image. Let's say, we need to use [Scikit learn](https://scikit-learn.org/stable/install.html), then the Docker image should contains [Scikit learn](https://scikit-learn.org/stable/install.html) library. Hence, the easiest way to make sure our Docker image has what we need is to set up a Conda environment with a yml file. 

In our project, we will be needed [Scikit learn](https://scikit-learn.org/stable/install.html), [Dask](https://dask.org/) and [XGBoost](https://pypi.org/project/xgboost/).  

## Create a local Anaconda environment

First of all, we will create a Conda environment with the following library:



```yml
python=3.8.3
  - numpy=1.18.1
  - pandas=1.0.3 
  - blosc=1.9.1
  - lz4=3.0.2
  - matplotlib
  - seaborn
  - dask-ml
  - jupyter
  - jupyterlab 
  - jupytext
  - s3fs
  - git
  - black
  - jupyterlab_code_formatter
  - conda-forge::scikit-learn=0.23.1
  - jupyterlab-git
  - dask-xgboost==0.1.10
  - pip:
    - dask-cloudprovider
    - git+git://github.com/thomaspernet/aws-python
    - awscli
    - nodejs
```



We specify the version in the yaml file to match the Docker image. If you run your own image, and more updated version are available, simple change the version.  I like to use Jupyter lab, with couple of extension like [jupyterlab_code_formatter](https://github.com/ryantam626/jupyterlab_code_formatter), [Git](https://anaconda.org/anaconda/git) and [Jupytext](https://github.com/mwouts/jupytext). On top, I use an [AWS wrapper](https://github.com/thomaspernet/aws-python) I wrote to access AWS data. You don't need them. Besides, we use the latest version of Python. 

The yaml file, named `dask_env.yml`  is available here.

Open the terminal, go to the directory with the yaml and run the following to create a conda environment with the libraries needed:

```yaml
conda create -n dask_env dask_env.yml ### Create a conda env named dask_en
```



To activate the environment, run `conda activate dash_env` .



## Create A docker images

Our second step consists in building the Docker image. if you are not familiar with Docker, please follow the official [tutorials](https://docs.docker.com/get-started/part1/). 

To make things simple, we can create the Docker image inside the same directory, in a subfolder named `Docker_image` 

To create a new folder, run `mkdir Docker_image` , and create a file without extension. Rename this file `Dockerfile` . We can use the recommended Docker image by [cloudprovider](https://github.com/dask/dask-cloudprovider). The image is available [[dask-docker](https://github.com/dask/dask-docker)]. However, we need to make some changes to operate the image. 



Here are some changes:

```dockerfile
python==3.8.3 \
    python-blosc \
    cytoolz \
    dask==2.16.0 \
    scikit-learn==0.23.1 \
    dask-ml \
    matplotlib \
    seaborn \
    lz4 \
    nomkl \
    numpy==1.18.1 \
    pandas==1.0.3 \
    tini==0.18.0 \
    dask-xgboost==0.1.10 \
    s3fs
```

Note that, I changed the ENTRYPOINT to allow permission when generating the machine in Fargate. Without the permission, Fargate cannot build the image

```dockerfile
ENTRYPOINT ["tini", "-g", "--","sh", "/usr/bin/prepare.sh"]
```

Do not forget to download and place the [shell script](https://github.com/dask/dask-docker/blob/master/base/prepare.sh) in the same directory as the Dockerfile.

the Docker image is now ready to be built. Make sure Docker runs on your machine and run the following command:

```shell
docker build --tag dask-container:py-38 .
```

The image is named `dask-container` and tagged `py-38` . 



## Push the Docker images on Docker Hub

if you want [cloudprovider](https://github.com/dask/dask-cloudprovider) to build the image on Fargate, you need to push the image on [Docker Hub](https://docs.docker.com/docker-hub/). Once again, if you are not familiar with it, read the tutorial [here](https://docs.docker.com/get-started/part3/). 

Since the image is already build, we need to tag it and push it to our public repositories

``` shell
docker tag dask-container:py-38 thomaspernet/dask-container:py-38
docker push thomaspernet/dask-container:py-38
```

Repeat step 2 and 3 for each change in the DockerFile. To check your running images, run `docker image ls` . 

You can visit my public images on [Docker Hub](https://hub.docker.com/repository/docker/thomaspernet/dask-container/tags?page=1).



## Run the cluster on Jupyter

In step 1, we created a Conda environment with the libraries we will need. In step 2 and 3, we created a Docker images and push it to Docker Hub. In this last step, we will connect our local machine to a Fargate cluster. I proceed in this way to save cost. You can actually do it in SageMaker directly. Refer to this [GitHub](https://github.com/rsignell-usgs/sagemaker-fargate-test) to use [cloudprovider](https://github.com/dask/dask-cloudprovider) in SageMaker. 

We created a notebook to detail the step to run Scikit learn with Dask without any issue. If you only want to connect to a cluster, it's simple, activate `dask_env`,  open a notebook and run this code

``` python
from dask_cloudprovider import FargateCluster
from dask.distributed import Client
cluster = FargateCluster(n_workers=4,
                         image='thomaspernet/dask-container:py-38'
                        )
client = Client(cluster)
client
```



It will provide a URL to follow the distributed computations on the cluster.

![](https://github.com/thomaspernet/Dask_cluster_aws_Docker/blob/master/notebooks_example/images/cluster.png?raw=true)

Change the image to your own repository.

- Notebook [machine-learning_cluster.ipynb](https://github.com/thomaspernet/Dask_cluster_aws_Docker/blob/master/notebooks_example/machine-learning_cluster.ipynb): Official Example
- Notebook [Dask_scikit_XGBoost](https://github.com/thomaspernet/Dask_cluster_aws_Docker/blob/master/notebooks_example/Dask_scikit_XGBoost.ipynb): Personal example

![](https://github.com/thomaspernet/Dask_cluster_aws_Docker/blob/master/notebooks_example/images/XGboost.gif?raw=true)

## Documentation

*Fargate and Dask*

- https://cloudprovider.dask.org/en/latest/
- https://github.com/rsignell-usgs/sagemaker-fargate-test
- https://medium.com/rapids-ai/getting-started-with-rapids-on-aws-ecs-using-dask-cloud-provider-b1adfdbc9c6e
- https://github.com/rsignell-usgs/dask-docker
- https://travis-ci.org/

*Docker*

- https://docs.docker.com/docker-hub/
- https://docs.docker.com/get-started/part2/
- https://docs.docker.com/get-started/part3/
- http://www.science.smith.edu/dftwiki/index.php/Tutorial:_Docker_Anaconda_Python_--_4
- https://docs.dask.org/en/latest/remote-data-services.html





