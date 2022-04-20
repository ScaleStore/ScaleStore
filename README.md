# ScaleStore: A Scalable Key-Value Store with Full Memory Disaggregation

This is the implementation repository of our submitted paper: **ScaleStore: A Scalable Key-Value Store with Full Memory Disaggregation**.



## Description

This artifact provides the source code of ***ScaleStore*** and scripts to reproduce the main experiments. *ScaleStore* is a fully memory-disaggregated KV store, which stores both data and metadata on memory nodes of DM to rule out monolithic servers. We address two critical challenges incurred by the weak-computation-power nature of the memory nodes, i.e., maintaining strongly consistent metadata replicas and handling corrupted metadata on client failures. To this end, *ScaleStore* proposes a sequentialization-free replication protocol to efficiently resolve conflicts and a two-level memory management scheme to avoid conflicts on the replicated metadata. *ScaleStore* proposes an embedded operation log that embeds log entries to KV pairs to recover and repair the metadata corruption without additional log maintenance overhead. We evaluate the performance on both micro-benchmarks and the YCSB hybrid benchmark. The results show that *ScaleStore* scales better than state-of-the-art KV stores on DM by up to 4.5 times.

You can access this artifact using the following command:
```shell
git clone https://github.com/ScaleStore/ScaleStore.git
```


## Environment

#### 1. Hardware dependencies
To reproduce the experiments, each machine should be equipped with one **8-core Intel processer**(*e.g.*, Intel Xeon E5-2450),  **16GB DRAM**  and one **RDMA NIC card** (*e.g.*, Mellanox ConnectX-3). Each RNIC should be connected to an **Infiniband or Ethernet switch** (*e.g.*, Mellanox SX6036G). All machines are separated into memory nodes and compute nodes. At maximum 5 memory nodes and 17 compute nodes are used for the experiments in our paper. If you do not have such testbed, consider using [CloudLab](https://www.cloudlab.us/).

#### 2. Software dependencies
* **Operating system:** Ubuntu 18.04 is recommended for each machine.

* **Compile toolchain:** g++ >= 7.5.0 and cmake >= 3.16.8

* **Other software dependencies:** Mellanox OFED, boost 1.65.1
 
* **HugePages setting:** 7168 HugePages of 2MB size in each memory node and 2048 ones in compute nodes is required. You can set up this with  `echo 7168 > /proc/sys/vm/nr_hugepages` command for memory nodes and `echo 2048 > /proc/sys/vm/nr_hugepages` for compute nodes.



## Configurations

Configuration files for servers and clients should be provided to the program. Here are two example configuration files below.

#### 1. Servers configuration

For each memory node, you should provide a configuration file `server_config.json`  where you can flexibly configure the server:

```json
{
    "role": "SERVER",
    "conn_type": "IB",
    "server_id": 0,
    "udp_port": 2333,
    "memory_num": 3,
    "memory_ips": [
        "10.10.10.1",
        "10.10.10.2",
        "10.10.10.3"
    ],
    "ib_dev_id": 0,
    "ib_port_id": 1,
    "ib_gid_idx": 0,

    "server_base_addr":  "0x10000000",
    "server_data_len":   15032385536,
    "block_size":        67108864,
    "subblock_size":     256,
    "client_local_size": 1073741824,

    "num_replication": 3,

    "main_core_id": 0,
    "poll_core_id": 1,
    "bg_core_id": 2,
    "gc_core_id": 3
}
```

For briefness, we call each memory node as "server `i`" (`i` = 0, 1, ...).

#### 2. Clients configuration

For each compute node, you should provide a configuration file `client_config.json` where you can flexibly configure the client:

```json
{
    "role": "CLIENT",
    "conn_type": "IB",
    "server_id": 2,
    "udp_port": 2333,
    "memory_num": 2,
    "memory_ips": [
        "128.110.96.102",
        "128.110.96.81"
    ],
    "ib_dev_id": 0,
    "ib_port_id": 1,
    "ib_gid_idx": 0,

    "server_base_addr":  "0x10000000",
    "server_data_len":   15032385536,
    "block_size":        67108864,
    "subblock_size":     1024,
    "client_local_size": 1073741824,

    "num_replication": 2,
    "num_idx_rep": 1,
    "num_coroutines": 10,
    "miss_rate_threash": 0.1,
    "workload_run_time": 10,
    "micro_workload_num": 10000,

    "main_core_id": 0,
    "poll_core_id": 1,
    "bg_core_id": 2,
    "gc_core_id": 3
}
```

For briefness, we call each compute node as "client `i`" (`i` = 0, 1, 2, ...).

It should be noted that, the `server_id` parameter of client `i` should be set to `2+i*8`. For example, the `server_id` of the first three client is 2, 10, 18 respectively.



## Experiments

For each node, execute the following commands to compile the entire program:

```shell
mkdir build && cd build
cmake ..
make -j
```

We test *ScaleStore* with **micro-benchmark** and **YCSB benchmarks** respectively. For each experiments, you should place `server_config.json` in directory `./build` on each memory nodes, and then execute the following command in memory nodes to set up servers:

```shell
numactl -N 0 -m 0 ./ycsb-test/ycsb_test_server [SERVER_NUM]
```

`[SERVER_NUM]` should be the serial number of this memory node, counting from 0.



#### 1. Micro-benchmark

* **Latency**

    To evaluate the latency of each operation, we use a single client to iteratively execute each operation (**INSERT**, **DELETE**, **UPDATE**, and **SEARCH**) for 10,000 times.

    Enter `./build/micro-test` and execute the following command in client `0`：

    ```shell
    numactl -N 0 -m 0 ./latency_test_client [PATH_TO_CLIENT_CONFIG]
    ```

    Test results will be saved in `./build/micro-test/results`.

* **Throughput**

    To evaluate the throughput of each operations, each client first iteratively INSERTs different keys for 0.5 seconds. UPDATE and SEARCH operations are then executed on these keys for 10 seconds. Finally, each client executes DELETE for 0.5 seconds.

    Enter `./build/micro-test` and execute the following command on all client nodes at the same time:

    ```shell
    numactl -N 0 -m 0 ./micro_test_multi_client [PATH_TO_CLIENT_CONFIG] 8
    ```

    Number `8` indicates there are 8 client threads in each client node. You will need to use the keyboard to simultaneously send space signals to each client node for starting each operation testing synchronously.

    Test results will be displayed on each client terminal.



#### 2. YCSB benchmarks 

* **Workload preparation**

    Firstly, download all the testing workloads using `sh download_workload.sh` in directory `./setup` and unpack the workloads you want into `./build/ycsb-test/workloads`.

    Here is the description of the YCSB workloads:

    | Workload | SEARCH | UPDATE | INSERT |
    | -------- | ------ | ------ | ------ |
    | A        | 0.5    | 0.5    | 0      |
    | B        | 0.95   | 0.95   | 0      |
    | C        | 1      | 0      | 0      |
    | D        | 0.95   | 0      | 0.05   |
    | upd[X]   | 1-[X]% | [X]%   | 0      |

    Then, you should execute the following command in `./build/ycsb-test` to split the workloads into N parts(N is the **total number** of client threads, that is, 8 * number of client nodes):

    ```shell
    python split-workload.py [N]
    ```

    And then we can  start testing *ScaleStore* using YCSB benchmarks.

* **Latency**

    To show the effectiveness of the **SNAPSHOT** replication protocol and the  **adaptive index cache** , we compare the operations latency of *ScaleStore* with two alternative implementations, *i.e.*, *ScaleStore-CR* and *ScaleStore-NC*. Here is the command of testing the latency of *ScaleStore*:

    ```shell
    numactl -N 0 -m 0 ./ycsb_latency_multi_client [PATH_TO_CLIENT_CONFIG] [WORKLOAD-NAME] 8
    ```

    Execute the command on all the client nodes at the same time. `[WORKLOAD-NAME]` can be chosen from `workloada`  to  `workloadd`.  Number `8` indicates there are 8 client threads in each client node. You will need to use the keyboard to simultaneously send space signals to each client node for starting each operation testing synchronously.

    Test results will be saved in `./build/ycsb-test/results`.

* **Throughput**

    To show the **scalability** of *ScaleStore*，we can test the throughput of *ScaleStore* with different number of client nodes. Besides, we can evaluate the **read-write performance** of *ScaleStore* by testing the throughput of *ScaleStore* using workloads with different search-update ratios `X`. Here is the command of testing the throughput of *ScaleStore*:

    ```shell
    numactl -N 0 -m 0 ./ycsb_test_multi_client [PATH_TO_CLIENT_CONFIG] [WORKLOAD-NAME] 8
    ```

    Execute the command on all the client nodes at the same time. `[WORKLOAD-NAME]` can be chosen from `workloada ~ workloadd` or `workloadudp0 ~ workloadudp100` (indicating different search-update ratios) .  Number `8` indicates there are 8 client threads in each client node. You will need to use the keyboard to simultaneously send space signals to each client node for starting each operation testing synchronously.

    Test results will be displayed on each client terminal.

