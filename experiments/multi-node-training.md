---
description: Running multi-worker training with Distribution Strategies.
---

# Multi-node Training

Note: Multi-node training is an advanced feature! For a primer, [read the TensorFlow documentation on Distribution Strategy](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/distribute/README.md#multi-worker-training) for multi-worker training.

Once you're ready, this article will walk you through a code recipe for performing multi-node training, as well as a sample command for running a multi-node experiment on Gradient.

{% hint style="info" %}
Multi-node training requires multiple concurrent jobs. Make sure your account subscription plan has a high enough max concurrent jobs for your needs.
{% endhint %}

### The Recipe

First we specify a cluster architecture with one master – `'ps'` – running on `192.168.1.1:1111` and two workers running on `192.168.1.2:1111` and `192.168.1.3:1111` respectively:

```text
import sys

import tensorflow as tf

# specify the cluster's architecture

cluster = tf.train.ClusterSpec({'ps': ['192.168.1.1:1111'], 'worker': ['192.168.1.2:1111','192.168.1.3:1111']})Copy
```

Note that the same code is replicated on multiple machines; therefore it's important to specify the role of the current execution node at the command line. A machine can be either a worker or a parameter server \("ps"\):

```text
# parse command-line to specify machine as defined in the ClusterSpec

job_type = sys.argv[1] # job type: "worker" or "ps" task_idx = sys.argv[2] # index job in the worker or ps listCopy
```

Run the training server, providing a cluster and specifying a role \(worker or ps\) and an id for each computational node:

```text
# create the TensorFlow Server (this is how the machines communicate)

server = tf.train.Server(cluster, job_name=job_type, task_index=task_idx)Copy
```

Note that computation will differ based on the role of the specific computational node:

If the role is a parameter server, then the condition is to join the server. In this case there is no code to execute because the workers will continuously push updates, so the only thing that the parameter server does is wait.

If the role is a worker, then the worker code is executed on those specific nodes within the cluster. This part of the code is similar to the code that we execute on a single machine when we first build the model and then train it locally. Note that all of the distribution of work and the collection of updated results is done transparently by TensorFlow.

TensorFlow provides a convenient `tf.train.replica_device_setter()` function that automatically assigns operations to devices:

```text
# parameter server is updated by remote clients.
# will not proceed beyond this if statement.
if job_type == 'ps':
server.join() else:
# workers only
with tf.device(tf.train.replica_device_setter( worker_device='/job:worker/task:'+task_idx, cluster=cluster)):
# build your model here as if you only were using a single machine
 
with tf.Session(server.target):
# train your model hereCopy
```

### How It Works

In the above example, we saw how to create a cluster with multiple computational nodes. A node can play the role of either a parameter server or a worker.

In either case, the same code is executed, but the execution of the code differs based on the parameters specified at the command line. The parameter server only needs to wait until the workers send updates.

Note that `tf.train.replica_device_setter(...)` is the function that assigns operations to available devices, while `tf.train.ClusterSpec(...)` is used for cluster setup.

## Modify your code to run distributed on Gradient

You can run the original Google mnist-sample code on Paperspace with minimal changes by simply setting `TF_CONFIG` and `model_dir` as follows.

**Set TF\_CONFIG environment variable**

First import from gradient-sdk:

```text
from gradient_sdk import get_tf_config
```

then in your main\(\):

```text
if __name__ == '__main__':
    get_tf_config()
```

This function will set `TF_CONFIG`, `INDEX` and `TYPE` for each node.

For multi-worker training, as mentioned above, you need to set the `TF_CONFIG` environment variable for each binary running in your cluster. The `TF_CONFIG` environment variable is a JSON string that specifies the tasks that constitute a cluster, each task's address, and each task's role in the cluster.

#### Set model\_dir 

The `model_dir` argument represents the directory where model parameters, graphs, etc. will be saved. This can also be used to load checkpoints \(from that directory\) into an estimator in order to continue training a previously saved model.

For multi-node scenarios on Gradient, please make sure to set it to:

```text
model_dir = os.path.abspath(os.environ.get('PS_MODEL_PATH'))
```

 You can also use gradient\_sdk:

```text
from gradient_sdk.utils import data_dir, model_dir, export_dir
```

And that's it! You can now run multi-worker scenarios on Gradient!

### Creating a multi-node experiment using the CLI

The following command creates and starts a multi-node experiment within the Gradient Project specified with the `--projectId` option. 

```bash
gradient experiments run multinode \
  --name my-multinode-mnist-experiment \
  --projectId <your-project-id> \
  --experimentEnv "{\"EPOCHS_EVAL\":5,\"TRAIN_EPOCHS\":10,\"MAX_STEPS\":1000,\"EVAL_SECS\":10}" \
  --experimentType GRPC \
  --workerContainer tensorflow/tensorflow:1.13.1-gpu-py3 \
  --workerMachineType K80 \
  --workerCommand 'pip install -r requirements.txt && python mnist.py' \
  --workerCount 2 \
  --parameterServerContainer tensorflow/tensorflow:1.13.1-py3 \
  --parameterServerMachineType K80 \
  --parameterServerCommand 'pip install -r requirements.txt && python mnist.py' \
  --parameterServerCount 1 \
  --workspaceUrl https://github.com/Paperspace/mnist-sample.git
```

We currently only support gRPC connections between the workers.

For more information about this sample experiment, see the README in the Paperspace [mnist-sample GitHub repo](https://github.com/Paperspace/mnist-sample). \(Note: the code for this experiment can be run in both singlenode and multinode training modes.\)

## 

