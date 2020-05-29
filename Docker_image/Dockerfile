FROM continuumio/miniconda3:4.7.12

RUN conda install --yes \
    -c conda-forge \
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
    s3fs \
    && conda clean -tipsy \
    && find /opt/conda/ -type f,l -name '*.a' -delete \
    && find /opt/conda/ -type f,l -name '*.pyc' -delete \
    && find /opt/conda/ -type f,l -name '*.js.map' -delete \
    && find /opt/conda/lib/python*/site-packages/bokeh/server/static -type f,l -name '*.js' -not -name '*.min.js' -delete \
    && rm -rf /opt/conda/pkgs

RUN pip install xgboost

COPY prepare.sh /usr/bin/prepare.sh

RUN mkdir /opt/app

ENTRYPOINT ["tini", "-g", "--","sh", "/usr/bin/prepare.sh"]