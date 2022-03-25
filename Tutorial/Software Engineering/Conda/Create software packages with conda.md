# Create software packages with conda


1. Create an account in <https://anaconda.org/> if you don’t have one.
2. Open command line terminal and login in your anaconda account by typing `anaconda login`
3. Install **conda build** with `conda install conda-build`
4. Create the **recipe** of your package, which is basically a folder with two files: **meta.yaml** and **build.sh** . **meta.yaml** file describe your package information like name, source code link, dependences *etc*, and **build.sh** is the command lines to install your package from source code. See [example](/doc/examples-BxyYoFjccG) for details.
5. With meta.yaml and build.sh ready, you can build the package with `conda build <the folder containing build.sh and meta.yaml>`.
6. If your package is not automatically uploaded to your channel in anaconda, follow the instruction of the output message of `conda build`.
7. login your anaconda account or search your package name in <https://anaconda.org/>, you will find your package. Click your package, you can see installation instruction.

   \


