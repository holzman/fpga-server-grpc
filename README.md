# MNIST on Google cloud: launching machine learning as a service
The purpose of this project is to provide a basis on which to stage machine learning services for use at the CMS experiment. It was programmed by Jack Dinsmore in the summer of 2019.

Specifically, this code will launch a gRPC server encoded in *server.py* on a Google Kubernetes Engine (GKE) cluster. You can interface with the server with any of the client programs in this repository.

# Description of server and docker files
- *.dockerignore*: a list of files to be excluded when creating an image of the repository to be run on the GKE.
- *_COMPILE.BAT*: a one-line executable which compiles *server-tools.proto*. *_COMPILE.BAT* should be run after modifying *server-tools.proto* for the changes to take effect. If you run linux, you can simply remove the `@echo off` command and it should run correctly.
- *Dockerfile*: a set of instructions read by `docker` on how to compile the repository to an image. The `RUN` commands contain all shell commands that should be run, such as those that install the python modules, and the `CMD` command actually starts the server on the GKE.
- *ml_functions.py*: a module which contains all of the machine learning aspects of the server. If you wish to use an implementation other than `keras`, change the neural net structure, or use a model other than MNIST, this would be the file to change. Currently, it runs MNIST with a CNN implementation in `keras` on the GPU.
- *mnist-cnn.h5*: a pretrained checkpoint for MNIST which the server uses for predictions
- *server.py*: the entrypoint for the server and the file which contains all of the code that interfaces with the client.
- *server_tools_pb2\**: files generated by *_COMPILE.BAT*. They should not be edited; instead, you should edit *server-tools.proto* and run *_COMPILE.BAT* again.
- *server-tools.proto*: a Java-like file that defines all the data structures used to shuttle data between the server and client over the internet


# Description of clients
**Please make sure to copy and paste the correct IP of the server into the file *IP.txt* in order for the client programs to work**
- *client-wait.py*: this client sends the information encoded in *image.bmp* to the server and asks it to produce a prediction. It halts execution until it receives an answer.
- *client-no-wait.py*: this client sends the information encoded in *image.bmp* to the server and asks it to produce a prediction. It does not halt execution after sending the data; instead it periodically checks in with the server to see if the prediction has been generated. This comes with a performance loss; time is wasted by asking the server whether the image has been processed yet.
- *client-big.py*: this client sends an arbitrary amount of data to the server and gets back a numpy array of predictions. You can specify whether the program should halt execution until it receives the predictions or continue execution, periodically checking in with the server to see if the predictions have been produced by changing the `WAIT` flag in the code
- *client-get-latency.py*: this client measures what fraction of time it takes to process data on the server is spent actually predicting. Ideally it would be zero every time, but it isn't. Some of the major causes of the latency are transmition time, time to convert the data and predictions from bytes to a numpy array, and time to set up threads on the server. We measure this fraction as a function of the number of images sent and whether the client waits for a response or continually checks in with the server to see if the prediction is done. 


# How to deploy the server
**Requirements:** You must have the `gcloud` command installed on your system. To do this, you may use (Google's tutorial) on how to install `gcloud` on your personal computer, or you can use the google cloud shell which you can get by clicking the **>_** icon in the upper right corner of the google cloud console. This should have `gcloud` installed. If you are using the cloud console, you must clone this repository to your system there.
1. Create a Google Cloud Kubernetes Cluster
    1. Open the menu in the upper left corner of the Google Cloud console and select **Kubernetes Engine>Clusters**. Then select **Create Cluster**. 
    2. Name the cluster whatever you want and change the region if you wish.
    3. Configure it with as many CPUs as you want. To add GPUs, click **Customize** in the box with the heading **default pool** and change the settings there.
    5. Click **create**.
2. Containerize this repository with docker
    1. Make any changes to the repository you wish to make.
    2. Run the shell command `gcloud builds submit --tag gcr.io/harrisgroup-223921/<<<NAME>>> .` where `<<<NAME>>>` is a name you pick for the container. This does not have to be the same name as the cluster that you created in the last step.
3. Deploy the server
    1. Click on the cluster you created in the first step. In the ribbon at the top, click **+ DEPLOY**. Keep **Existing Container Image** selected and click **SELECT** to the right of the box beneath. From the menu that pops up, select the container you created in step two; it should show up as `<<<NAME>>>`.
    2. Click Continue.
    3. Choose a fitting name for your deployment.
    4. Confirm that the name of your cluster is selected in the dropdown menu at the bottom of the page. 
    5. Click **DEPLOY**.
4. Expose the server (Also called deploying the service)
    1. Click **Expose** in one of the messages that appears at the top of your deployment if it succeeds. If the message is not there, click **ACTIONS** in the ribbon at the top and select **Expose**.
    2. Choose a port of **50051** and click **EXPOSE**
    3. Copy the IP listed in the **External Endpoints** field if your exposure succeeds.
5. Run whatever client programs you want
6. Clean up
    1. Delete the cluster.
    2. Delete the container you built in step 2 with `gcloud container images delete gcr.io/harrisgroup-223921/<<<NAME>>>`.

# Rules for changing the server
The server can be fairly freely without affecting the deployment procedure. However, there are a few rules that must be observed to avoid errors in deployment.

1. If you modify the *.proto* file, run *_COMPILE.BAT*. 
2. If you import a model named `<<<MODULE>>>` that has to be installed with pip, add the command `RUN pip install <<<MODULE>>>` to *Dockerfile*.
3. If you change the name of *server.py* or change the entrypoint of the server to something else, adjust the last line of *Dockerfile* to properly reflect the new file name.
4. If you add files to the repository which are not necessary for running the server, exclude them from the `docker` image by adding them to *.dockerignore* (This step might not cause errors; I have always followed it for form's sake and I don't know if it's necessary)
5. Do not change the *_pb2* python files. Change the *.proto* file instead.

# Error debugging
Even when the server and clients run perfectly on localhost, sometimes deployment to the GKE fails. The causes of some very common errors are listed below.
### Deployment errors
Deployment errors are listed at the top of the deployment page
- **Unschedulable**: this error can occur in deployments when *Dockerfile* has a typo in it, especially the `CMD` line. Sometimes it also happens if your cluster was set up incorrectly, so deleting and recreating your cluster may resolve the problem if you're sure *Dockerfile* is correct. 
- **Does not have minimum availability**: this is a placeholder error. It occurs when your cluster runs out of storage, but it also occurs in situations that have nothing to do with storage. For example, if your server has a bug in it, the pods will throw **CrashLoopBackOff** and your deployment will throw **Does not have minimum availability**. So check your pod logs for tracebacks before changing the storage size on your cluster.
- **Insufficient cpu**: another placeholder error. This error often accompanies **Unschedulable**, and once again, the problem often has nothing to do with the amount of processing power available.

### Pod errors
Pod errors are listed next to the pod name on the deployment page.
- **CrashLoopBackOff**: this error occurs in pods when the server cannot be initialized. To read the error, click on one of the pods which shows the **CrashLoopBackOff** error, go to the **Logs** page, and click on **Container Logs**. The traceback should be there.