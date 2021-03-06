# Distributed TensorFlow (Supervisor) 实践
- 摘要：在之前远程调用TensorFlow的基础上继续进行了探索，实现了多机共同协作完成一个任务的目标。
## 相关知识
- gRPC：google remote procedure call，分布式Tensorflow底层的通信协议。是一个远程过程调用。本地代码执行过程中，客户端将参数打包发送给服务器，服务器运行计算过程，最后将结果打包返回给客户端，给人以代码就在本地运行的感觉。
- Task：主机上的一个进程,在大多数情况下,一个机器上只运行一个Task。
- Job：Tasks的集合，一般把Job划分为Parameter和Worker,Parameter Job是管理参数的存储和更新工作。Worker Job是来运行ops。如果参数的数量太大,一台机器处理不了,这就要需要多个Jobs。
- Cluster：Jobs的集合，俗称集群系统。
## 实践过程
### 创建集群
本次实验过程中，使用了3台设备（192.168.73.130，192.168.73.132，192.168.73.133）来搭建集群，将192.168.73.130作为Parameter Job（参数服务器），其余两台作为Worker Job（计算服务器）。构建集群的相关代码如下：
```python
import tensorflow as tf

flags = tf.app.flags
flags.DEFINE_integer("train_steps", 2000, "Number of global training steps to perform")
flags.DEFINE_float("learning_rate", 0.01, "Learning rate")
flags.DEFINE_string("ps_hosts","192.168.73.130:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("worker_hosts", "192.168.73.132:2222, 192.168.73.133:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("log_dir", "./log/", "Log directory")
flags.DEFINE_string("job_name", None, "Job name: worker or ps")
flags.DEFINE_integer("task_index", None, "Worker task index, should be >= 0.")
FLAGS = flags.FLAGS

# construct the cluster
ps_spec = map(lambda str: str.strip(), FLAGS.ps_hosts.split(","))
worker_spec = map(lambda str: str.strip(), FLAGS.worker_hosts.split(","))
cluster = tf.train.ClusterSpec({ "ps": ps_spec, "worker": worker_spec})
```
### 定义Server
创建好了cluster，我们就需要在每台主机上定义各自的Server，同时指定此Task的Job_name和task_index，相关代码如下：
```python
# server
server = tf.train.Server(cluster, job_name=FLAGS.job_name, task_index=FLAGS.task_index)
```
而此段代码实际上在192.168.73.133主机上的运行效果是：
```python
server = tf.train.Server(cluster, job_name="worker", task_index=1)
```
其余两台主机类似。<br>
`tf.train.Server()`的作用是创建一个服务器，监听port端口，如果有数据传过来，它就会调用本地设备执行运算，然后将结果返回给调用者。<br>
都知道，server定义完成之后，因为Parameter Job不参与计算过程，所以它会直接结束，但是我们要让其一直处于监听状态，所以会在Parameter Job上做以下操作：
```python
# ps
if FLAGS.job_name == "ps":
    server.join()
```
对于Worker Job节点，其工作方式有两种In-graph replication和Between-graph replication：
- In-graph replication：一个client(显式调用`tf.Session()`的进程),将里面的参数和ops指定给对应的job去完成.数据分发只由一个client完成。
- Between-graph replication：很多独立的client，各个client构建了相同的graph（包含参数），通过使用`tf.train.replica_device_setter()`，将这些参数映射到Parameter Job上。

而对于训练过程，又分为同步训练和异步训练两种：
- 同步训练（Synchronous training）：在这种方式中，每个graph的副本读取相同的parameter的值，并行的计算gradients，然后将所有计算完的gradients放在一起处理。Tensorlfow提供了函数`tf.train.SyncReplicasOptimizer()`来处理这个问题。
- 异步训练（Asynchronous training）：每个client自己计算完gradient就去更新paramenter，不同replica之间不会去协调进度。

本次实验采用Between-graph replication + Synchronous training的训练模式。
### 定义模型
本次实验使用的是之前单机测试的逻辑回归的模型：
```python
with tf.device(tf.train.replica_device_setter(
    worker_device="/job:worker/task:%d" % FLAGS.task_index,
    cluster=cluster)):
    global_step = tf.Variable(0, name='global_step', trainable=False)

    # tensorFlow graph input
    X = tf.placeholder(tf.float32)
    Y = tf.placeholder(tf.float32)

    # random model weight
    W = tf.Variable(np.random.randn())
    b = tf.Variable(np.random.randn())

    # construct a linear model
    activation = tf.add(tf.multiply(X, W), b)

    # loss
    loss = tf.square(Y - activation)
    optimizer = tf.train.GradientDescentOptimizer(FLAGS.learning_rate)

    # use the sync_replicas mode
    opt = tf.train.SyncReplicasOptimizer(
        optimizer, 
        replicas_to_aggregate=len(worker_spec),
        total_num_replicas=len(worker_spec),
        use_locking=True,
        name="lr_sync_replicas")
    train_op = opt.apply_gradients(optimizer.compute_gradients(loss), global_step=global_step)

    # initial token and chief queue runners required by the sync_replicas mode
    chief_queue_runner = opt.get_chief_queue_runner()
    sync_init_op = opt.get_init_tokens_op()
```
### 创建Supervisor
`tf.train.Supervisor()`提供了一系列服务来帮助实现一个鲁棒的训练过程。主要的服务包括：
- 自动去checkpoint加载数据或初始化数据。
- 自带一个Saver，用来保存checkpoint。
- 自带一个summary_computed，用来保存Summary，使得任务可以在shutdown之后进行恢复。

创建的相关代码如下：
```python
# Supervisor
if FLAGS.task_index == 0:
    local_init_op = opt.chief_init_op
    is_chief = True
else:
    local_init_op = opt.local_step_init_op
    is_chief = False
ready_for_local_init_op = opt.ready_for_local_init_op
sv = tf.train.Supervisor(
    is_chief=is_chief, 
    logdir=FLAGS.log_dir, 
    ready_for_local_init_op=ready_for_local_init_op,
    local_init_op=local_init_op,
    init_op=tf.global_variables_initializer(), 
    recovery_wait_secs=1, 
    global_step=global_step)
```
### 执行训练
训练开始时，作为chief的worker job会创建session，而作为非chief的worker job则会等待chief的session创建完成，如下：
```python
# preform training
if FLAGS.task_index == 0:
    server_grpc_url = server.target
    print("Worker 0: Initializing session at %s..." % server_grpc_url)
else:
    server_grpc_url = "grpc://%s" % worker_spec[0]
    print("Worker %d: Waiting for session to be initialized at %s..." % (FLAGS.task_index, server_grpc_url))
sess_config = tf.ConfigProto(
	allow_soft_placement=True, 
	log_device_placement=False, 
	device_filters=["/job:ps", "/job:worker/task:%d" % FLAGS.task_index])
with sv.prepare_or_wait_for_session(
    server_grpc_url,
    config=sess_config
    ) as sess:
    print("Worker %d: Session initialization complete." % FLAGS.task_index)
```
创建完成session之后，chief节点还需要做一些初始化的工作：
```python
if FLAGS.task_index == 0:
    # Chief worker will start the chief queue runner and call the init op.
    sess.run(sync_init_op)
    sv.start_queue_runners(sess, [chief_queue_runner])
```
至此，就可以正式启动训练了：
```python
local_step = 0
while True:
    train_x = np.random.randn(1)
    train_y = 2 * train_x + np.random.randn(1) * 0.33  + 10
    _, _, step = sess.run([train_op, loss, global_step], feed_dict={X: train_x, Y: train_y})
    local_step += 1
    if step % 100 == 0:
        print("Worker %d: local_step=%d, global_step=%d, W=%.9f, b=%.9f" % (FLAGS.task_index, local_step, step, sess.run(W), sess.run(b)))
    if step >= FLAGS.train_steps:
        break
print("Optimization Finished: W=%.9f, b=%.9f" % (sess.run(W), sess.run(b)))
```
### 调用方式
调用方式如下（顺序无关）：
- 192.168.73.130（ps0）：`python distribute_lr.py --job_name="ps" --task_index=0`
- 192.168.73.132（worker0/chief）：`python distribute_lr.py --job_name="worker" --task_index=0`
- 192.168.73.133（worker1）：`python distribute_lr.py --job_name="worker" --task_index=1`
### 运行结果
192.168.73.130（ps0）启动程序，监听端口：
```text
2017-10-28 09:44:43.955646: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> localhost:2222}
2017-10-28 09:44:43.955691: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> 192.168.73.132:2222, 1 -> 192.168.73.133:2222}
2017-10-28 09:44:43.972714: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:2222
```
192.168.73.132（worker0/chief）启动程序，等待所有设备就绪后，初始化Session并开始训练：
```text
2017-10-28 09:44:51.451726: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> 192.168.73.130:2222}
2017-10-28 09:44:51.451769: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> localhost:2222, 1 -> 192.168.73.133:2222}
2017-10-28 09:44:51.465028: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:2222
2017-10-28 09:44:51.542812: I tensorflow/core/distributed_runtime/master_session.cc:998] Start master session bdd69f29f5d56fa3 with config: device_filters: "/job:ps" device_filters: "/job:worker/task:1" allow_soft_placement: true
Worker 0: Initializing session at grpc://localhost:2222...
2017-10-28 09:44:51.590253: I tensorflow/core/distributed_runtime/master_session.cc:998] Start master session d2a97c61672efedc with config: device_filters: "/job:ps" device_filters: "/job:worker/task:0" allow_soft_placement: true
Worker 0: Session initialization complete.
Worker 0: local_step=1, global_step=0, W=0.251293659, b=-0.250558496
Worker 0: local_step=2, global_step=0, W=0.347586691, b=-0.028887212
Worker 0: local_step=202, global_step=100, W=1.663281679, b=8.638786316
Worker 0: local_step=203, global_step=100, W=1.696257710, b=8.661085129
2017-10-28 09:44:52.604023: I tensorflow/core/distributed_runtime/master_session.cc:998] Start master session d92077259fb0ab24 with config: device_filters: "/job:ps" device_filters: "/job:worker/task:1" allow_soft_placement: true
Worker 0: local_step=362, global_step=200, W=1.927383900, b=9.835911751
Worker 0: local_step=462, global_step=300, W=1.962572813, b=9.989402771
Worker 0: local_step=562, global_step=400, W=2.010452509, b=9.971201897
Worker 0: local_step=662, global_step=500, W=1.996653080, b=9.994122505
Worker 0: local_step=762, global_step=600, W=1.992730021, b=9.978012085
Worker 0: local_step=862, global_step=700, W=2.012828112, b=9.976252556
Worker 0: local_step=962, global_step=800, W=2.029024124, b=10.024283409
Worker 0: local_step=1062, global_step=900, W=1.974614143, b=10.031742096
Worker 0: local_step=1162, global_step=1000, W=1.992137909, b=9.996970177
Worker 0: local_step=1262, global_step=1100, W=2.028121710, b=10.014409065
Worker 0: local_step=1362, global_step=1200, W=2.003846169, b=10.000977516
Worker 0: local_step=1462, global_step=1300, W=1.995067954, b=9.992860794
Worker 0: local_step=1562, global_step=1400, W=2.021365404, b=10.016706467
Worker 0: local_step=1662, global_step=1500, W=1.977930307, b=9.959195137
Worker 0: local_step=1762, global_step=1600, W=1.982629657, b=9.969757080
Worker 0: local_step=1862, global_step=1700, W=1.966497302, b=10.003686905
Worker 0: local_step=1962, global_step=1800, W=2.013534784, b=10.004679680
Worker 0: local_step=2062, global_step=1900, W=2.038592815, b=9.957181931
Worker 0: local_step=2162, global_step=2000, W=2.037247896, b=10.024839401
Optimization Finished: W=2.037247896, b=10.024839401
```
192.168.73.133（worker1）：启动程序，等待chief初始化Session后，开始训练：
```text
2017-10-28 09:44:48.812681: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> 192.168.73.130:2222}
2017-10-28 09:44:48.812722: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> 192.168.73.132:2222, 1 -> localhost:2222}
2017-10-28 09:44:48.826327: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:2222
Worker 1: Waiting for session to be initialized at grpc://192.168.73.132:2222...
Worker 1: Session initialization complete.
Worker 1: local_step=42, global_step=200, W=1.927383900, b=9.835911751
Worker 1: local_step=142, global_step=300, W=1.962572813, b=9.989402771
Worker 1: local_step=242, global_step=400, W=2.010452509, b=9.971201897
Worker 1: local_step=342, global_step=500, W=1.996653080, b=9.994122505
Worker 1: local_step=442, global_step=600, W=1.992730021, b=9.978012085
Worker 1: local_step=542, global_step=700, W=2.012828112, b=9.976252556
Worker 1: local_step=642, global_step=800, W=2.029024124, b=10.024283409
Worker 1: local_step=742, global_step=900, W=1.974614143, b=10.031742096
Worker 1: local_step=842, global_step=1000, W=1.992137909, b=9.996970177
Worker 1: local_step=942, global_step=1100, W=2.028121710, b=10.014409065
Worker 1: local_step=1042, global_step=1200, W=2.003846169, b=10.000977516
Worker 1: local_step=1142, global_step=1300, W=1.995067954, b=9.992860794
Worker 1: local_step=1242, global_step=1400, W=2.021365404, b=10.016706467
Worker 1: local_step=1342, global_step=1500, W=1.977930307, b=9.959195137
Worker 1: local_step=1442, global_step=1600, W=1.982629657, b=9.969757080
Worker 1: local_step=1542, global_step=1700, W=1.966497302, b=10.003686905
Worker 1: local_step=1642, global_step=1800, W=2.013534784, b=10.004679680
Worker 1: local_step=1742, global_step=1900, W=2.038592815, b=9.957181931
Worker 1: local_step=1842, global_step=2000, W=2.037247896, b=10.024839401
Optimization Finished: W=2.037247896, b=10.024839401
```
### 对比
单机与集群对比：
- 模型：均为逻辑回归
- 训练次数：均为4000
- 学习率：均为0.01
- 训练集：均为4000个随机点对
- 运行环境：单机与集群每个节点相同

最终对比结果：**单机：耗时12.56s；集群：耗时8.45s**
### 总结
本次对分布式TensorFlow的探索，**实现了多机共同协作完成一个任务的目标。并且初步验证了集群的确比单机训练速度更快的猜想**。但是由于训练模型简单，数据集很小，亦不可轻易下结论，在后面的研究中，将在svhn（The Street View House Numbers）Dataset上进行单机与集群、In-Graph与Between-Graph、同步和异步之间更为深入的对比。
### 附：完整代码
```python
import sys
import os
import shutil
import numpy as np
import tensorflow as tf

flags = tf.app.flags
flags.DEFINE_integer("train_steps", 2000, "Number of global training steps to perform")
flags.DEFINE_float("learning_rate", 0.01, "Learning rate")
flags.DEFINE_string("ps_hosts","192.168.73.130:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("worker_hosts", "192.168.73.132:2222, 192.168.73.133:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("log_dir", "./log/", "Log directory")
flags.DEFINE_string("job_name", None, "Job name: worker or ps")
flags.DEFINE_integer("task_index", None, "Worker task index, should be >= 0.")
FLAGS = flags.FLAGS

def main(argv):
    # remove old log directory
    if os.path.exists(FLAGS.log_dir):
        shutil.rmtree(FLAGS.log_dir)
    os.mkdir(FLAGS.log_dir)

    # check flags
    if FLAGS.job_name is None or FLAGS.job_name == "" or (FLAGS.job_name != "ps" and FLAGS.job_name != "worker"):
        raise ValueError("Must specify an explicit `job_name`, one of 'ps' and 'worker'")
    if FLAGS.task_index is None or FLAGS.task_index =="" or FLAGS.task_index < 0:
        raise ValueError("Must specify an explicit `task_index`")
    print("job name = %s" % FLAGS.job_name)
    print("task index = %d" % FLAGS.task_index)

    # construct the cluster
    ps_spec = map(lambda str: str.strip(), FLAGS.ps_hosts.split(","))
    worker_spec = map(lambda str: str.strip(), FLAGS.worker_hosts.split(","))
    cluster = tf.train.ClusterSpec({ "ps": ps_spec, "worker": worker_spec})

    # server
    server = tf.train.Server(
        cluster,
        job_name=FLAGS.job_name,
        task_index=FLAGS.task_index)

    # ps
    if FLAGS.job_name == "ps":
        server.join()
        return

    # worker
    with tf.device(tf.train.replica_device_setter(
        worker_device="/job:worker/task:%d" % FLAGS.task_index,
        cluster=cluster)):
        global_step = tf.Variable(0, name='global_step', trainable=False)

        # tensorFlow graph input
        X = tf.placeholder(tf.float32)
        Y = tf.placeholder(tf.float32)

        # random model weight
        W = tf.Variable(np.random.randn())
        b = tf.Variable(np.random.randn())

        # construct a linear model
        activation = tf.add(tf.multiply(X, W), b)

        # loss
        loss = tf.square(Y - activation)
        optimizer = tf.train.GradientDescentOptimizer(FLAGS.learning_rate)

        # use the sync_replicas (synchronized replicas) mode, where in the parameter updates from workers are aggregated before applied to avoid stale gradients
        opt = tf.train.SyncReplicasOptimizer(
            optimizer, 
            replicas_to_aggregate=len(worker_spec),
            total_num_replicas=len(worker_spec),
            use_locking=True,
            name="lr_sync_replicas")
        train_op = opt.apply_gradients(optimizer.compute_gradients(loss), global_step=global_step)

        # initial token and chief queue runners required by the sync_replicas mode
        chief_queue_runner = opt.get_chief_queue_runner()
        sync_init_op = opt.get_init_tokens_op()

    # Supervisor
    if FLAGS.task_index == 0:
        local_init_op = opt.chief_init_op
        is_chief = True
    else:
        local_init_op = opt.local_step_init_op
        is_chief = False
    ready_for_local_init_op = opt.ready_for_local_init_op
    sv = tf.train.Supervisor(
        is_chief=is_chief, 
        logdir=FLAGS.log_dir, 
        ready_for_local_init_op=ready_for_local_init_op,
        local_init_op=local_init_op,
        init_op=tf.global_variables_initializer(), 
        recovery_wait_secs=1, 
        global_step=global_step)

    # preform training
    if FLAGS.task_index == 0:
        server_grpc_url = server.target
        print("Worker 0: Initializing session at %s..." % server_grpc_url)
    else:
        server_grpc_url = "grpc://%s" % worker_spec[0]
        print("Worker %d: Waiting for session to be initialized at %s..." % (FLAGS.task_index, server_grpc_url))
    sess_config = tf.ConfigProto(allow_soft_placement=True, log_device_placement=False, device_filters=["/job:ps", "/job:worker/task:%d" % FLAGS.task_index])
    with sv.prepare_or_wait_for_session(
        server_grpc_url,
        config=sess_config
        ) as sess:
        print("Worker %d: Session initialization complete." % FLAGS.task_index)
        if FLAGS.task_index == 0:
            # Chief worker will start the chief queue runner and call the init op.
            sess.run(sync_init_op)
            sv.start_queue_runners(sess, [chief_queue_runner])
        local_step = 0
        while True:
            train_x = np.random.randn(1)
            train_y = 2 * train_x + np.random.randn(1) * 0.33  + 10
            _, _, step = sess.run([train_op, loss, global_step], feed_dict={X: train_x, Y: train_y})
            local_step += 1
            if step % 100 == 0:
                print("Worker %d: local_step=%d, global_step=%d, W=%.9f, b=%.9f" % (FLAGS.task_index, local_step, step, sess.run(W), sess.run(b)))
            if step >= FLAGS.train_steps:
                break
        print("Optimization Finished: W=%.9f, b=%.9f" % (sess.run(W), sess.run(b)))

if __name__ == '__main__':
    tf.app.run()
```
