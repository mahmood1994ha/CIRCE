# CIRCE - CentralIzed Runtime sChedulEr

# Introduction

CIRCE is a runtime scheduling software tool for dispersed computing, which can deploy pipelined
computations described in the form of a directed acyclic graph (DAG) on multiple geographically
dispersed computers (compute nodes).

The tool is run on a host node (also called scheduler node). It needs the information about 
compute nodes are available (such as IP address, username and password), the description of the 
DAG along with code for the corresponding tasks. Based on measurements of computation costs for each task on each
node and the communication cost of transferring output data from one node to another, it first uses a
DAG-based static scheduling algorithm (at present, we include a modified version of an implementation [2]
of the well-known HEFT algorithm [1] with the tool) to determine
at which node to place each task from the DAG. CIRCE then deploys the each tasks on the corresponding node, 
using input and output queues for pipelined execution and taking care of the data transfer between different nodes.

We use here Distributed Network Anomaly Detection application (DNAD) : https://github.com/ANRGUSC/DNAD

List of nodes for the experiment, including the scheduler node, is
kept in file nodes.txt (the user needs to fill the file with the appropriate IP addresses,
usernames and passwords of their compute nodes):

| scheduler | IP |username | pw |
| ------ |----|--|-- |
| node1 | IP| username |pw |
| node2 | IP| username| pw |
| node3  | IP |username |pw |

DAG description of DAND (as adjacency list, where the first item is a parent task and subsequent items are child tasks) is kept in two files dag.txt (used by HEFT):

| local_pro aggregate0 aggregate1 aggregate2 |
| ------ |
| aggregate0 simple_detector0 astute_detector0 |
| aggregate1 simple_detector1 astute_detector1 |
| aggregate2 1 true simple_detector2 astute_detector2  |
| simple_detector0 fusion_center0 |
| simple_detector1 fusion_center1 |
| simple_detector2 fusion_center2 |
| astute_detector0 fusion_center0 |
| astute_detector1 fusion_center1 |
| astute_detector2 fusion_center2 |
| fusion_center0 global_fusion |
| fusion_center1 global_fusion |
| fusion_center2 global_fusion |
| global_fusion scheduler |

and configuration.txt (config_security.txt in this example), that is of the form:

| 14 |
|-----|
| local_pro 1 false aggregate0 aggregate1 aggregate2 |
| aggregate0 1 true simple_detector0 astute_detector0 |
| aggregate1 1 true simple_detector1 astute_detector1 |
| aggregate2 1 true simple_detector2 astute_detector2 |
| simple_detector0 1 true fusion_center0 |
| simple_detector1 1 true fusion_center1 |
| simple_detector2 1 true fusion_center2 |
| astute_detector0 1 true fusion_center0 |
| astute_detector1 1 true fusion_center1 |
| astute_detector2 1 true fusion_center2 |
| fusion_center0 2 true global_fusion |
| fusion_center1 2 true global_fusion |
| fusion_center2 2 true global_fusion |
| global_fusion 3 true scheduler |

The first line is an integer, which gives the number of lines the DAG is taking in the file. DAG is represented in the form of adjacency list as before (parent_task child_task1 child_task2 child task3 ...) but with two additional parameters:

parent_task NUM_INPUTS FLAG child_task1 child_task2 child task3 ...

NUM_INPUTS is an integer. It represents the number of input files the task needs in order to start processing (some tasks could require more than input).

FLAG is ‘true’ or ‘false’. Based on its value, monitor.py will either send a single output of the task to all its children (when true), or it will wait the output files and start putting them into queue (when false). Once the queue size is equal to the number of children, it will send one output to one child (first output to first listed child, etc.).

HEFT will append its output to this file, which is the input to centralized run-time scheduler.


# User Guide
The system consists of several tools and requires the following steps:

- PROFILING:
  - Execution profiler: produces profiler_nodeX.txt file for each node,
    which gives the execution time of each task on that node and the
    amount of data it passes to its child tasks. These results are required
    in the next step for HEFT algorithm.
    - INPUT: dag.txt, nodes.txt, DAG task files (task1.py, task2.py,. . . ), DAG input file (input.txt)
    - OUTPUT: profiler_nodeNUM.txt
    - USER GUIDE: There are two ways to run the execution profiler:
        1. copy the app/ folder to the each of the nodes using scp and
           inside app/ folder perform the following commands:
           ```sh
           $ docker build –t profilerimage .
           $ docker run –h hostname profilerimage
           ```
           where hostname is the name of the node (node1, node2, etc.,..).
        2. inside circe/docker_execution_profiler/ folder
           perform the following command:
           ```sh
           $ python3 scheduler.py
           ```
           In this case, the file scheduler.py will copy the app/ folder
           to each of the nodes and execute the docker commands.
           In both cases make sure that the command inside file app/start.sh
           gives the details (IP, username and password) of your scheduler machine.
  - Central network profiler: automatically scheduling and logs com-
    munication information of all links betweet nodes in the network,
    which gives the quaratic regression parameters of each link repre-
    senting the corresponding communication cost. These results are
    required in the next step for HEFT algorithm.
    - INPUT: central.txt stores credential information of the central node:
        | CENTRAL IP     | USERNAME |  PASSWORD |
        | -------------- | -------- | --------  |
        | IP0            | USERNAME |  PASSWORD |

        nodes.txt stores credential information of the nodes information:

        |TAG    |  NODE (USERNAME@IP)     | REGION  | PASSWORD  |
        |-----  |  ---------------------  | ------  | --------  |
        |node1  |  USERNAME1@IP1          | LOC1    | PASSWORD1 |
        |node2  |  USERNAME2@IP2          | LOC2    | PASSWORD2 |
        |node3  |  USERNAME3@IP3          | LOC3    | PASSWORD3 |

        link list.txt stores the the links between nodes required to log
        the communication.

        |SOURCE(TAG) |   DESTINATION(TAG)   |
        |----------- |   ----------------   |
        |node1       |   node2              |
        |node1       |   node3              |
        |node2       |   node1              |
        |node2       |   node3              |
        |node3       |   node1              |
        |node3       |   node2              |

    - OUTPUT: all quadratic regression parameters are stored in the
        local MongoDB on the central node.

    - USER GUIDE AT CENTRAL NETWORK PROFILER:

            1. run the command ./central init to install required libraries
            2. inside the folder central input add information about the
            nodes and the links.
            3. python3 central scheduler.py to generate the schedul-
            ing files for each node, prepare the central database and col-
            lection, copy the scheduling information and network scripts
            for each node in the node list and schedule updating the
            central database every 10th minute.

    - USER GUIDE AT OTHER NODES:

        1. The central network profiler copied all required scheduling
            files and network scripts to the folder online profiler in each
            compute node (droplet).
        2. run the command ./droplet init to install required libraries
        3. run the command python3 automate droplet.py to gen-
           erate files with different sizes to prepare for the logging mea-
           surements, generate the droplet database, schedule logging
           measurement every minute and logging regression every 10th
           minute.

  - System resource profiler:This tool will get system utilization from
    node 1, node 2 and node 3. Then these information will be sent to
    scheduler node and stored into mongoDB.The information includes:
    IP address of each node, cpu utilization of each node, memory uti-
    lization of each node, and the latest update time.
    - USER GUIDE:
      For working nodes: copy the mongo script/ folder to each
      working node using scp. In each node, type:
      ```sh
      $ python2 mongo script/install package.py
      $ python2 mongo script/server.py
      ```
      For scheduler node: copy mongo control/ folder to scheduler node using scp under circe/central_network profiler if a node’s IP address changes, just update the mongo control/ip path file inside apac scheduler/central network profiler/mongo control/ folder, type: python2 install package.py
            python2 jobs.py (if you want to run in backend, type: python2 jobs.py and then close the                terminal)

- HEFT (adapted/modified from [2])
  - HEFT input file construction: HEFT implementation takes a file of .tgff format, which describes the DAG and its various costs, as input. The first step is to construct this file.
    - INPUT: dag.txt, profiler_nodeNUM.txt
    - OUTPUT: input.tgff
    - USER GUIDE: from circe/heft/ folder execute:
    ```sh
     $ python write_input_file.py
    ```
  - HEFT algorithm. This is the scheduling algorithm which decides
    where to run each task. It writes its output in a configuration file,
    needed in the next step by the run-time centralized scheduler.
    - INPUT: input.tgff
    - OUTPUT: configuration.txt
    - USER GUIDE: from circe/heft/ run:
    ```sh
    $ python main.py
    ```

- CENTRALIZED SCHEDULER WITH PROFILER
  - Centralized run-time scheduler. This is the run-time scheduler. It takes the
    configuration file, given by HEFT, and orchestrates the execution of
    tasks on given nodes.
    - INPUT: configuration.txt, nodes.txt
    - OUTPUT: DAG output files appear in circe/centralized_scheduler/output/ folder
    - USER GUIDE: inside circe/centralized_scheduler/ folder run:
    ```sh
    $ python3 scheduler.py
    ```
    wait several seconds and move input1.txt to apac scheduler/centralized_scheduler/input/
    folder (repeat the same for other input files).
  - Stopping the centralized run-time scheduler.  Run:
    ```
    $ python3 removeprocesses.py
    ```
    This script will shh into every node and kill running processes, and kill the process on the master node.
    
  - If network conditions change, one might want to restart the whole application. This can be done by running:
    ```
    $ python3 remove_and_restart.py
    ```
    The first part of the script stops the system as described above. It then runs HEFT and restarts the centralized run-time scheduler with the new task-node mapping.
  - Run-time task profiler


# Project Structure

It is assumed that the folder circe/ is located on the users home path
(for example: /home/apac). The structure of the project within circe/
folder is the following:

- nodes.txt
- dag.txt
- configuration.txt (output of the HEFT algorithm)
- profiler node1.txt, profiler node2.txt,... (output of execution profiler)
- docker_execution_profiler/
    - scheduler.py
    - app/
        - dag.txt
        - requirements.txt
        - Dockerfile
        - DAG task files (task1.py, task2.py,...)
        - DAG input file (input1.txt)
        - start.sh
        - profiler.py
- centralized scheduler with profiler/
    - input/ (this folder should be created by user)
    - output/ (this folder should be created by user)
    - 1botnet.ipsum, 2botnet.ipsum (example input files)
    - scheduler.py
    - monitor.py
    - securityapp (this folder contains application task files, in this case localpro.py, aggregate0.py,...)
    - removeprocesses.py
    - remove_and_restart.py
    - readconfig.py
- heft/
    - write_input_file.py
    - heft_dup.py
    - main.py
    - create_input.py
    - cpop.py
    - read config.py
    - input.tgff (output of write input file.py)
    - readme.md
- central network profiler/
    - folder central_input: link list.txt, nodes.txt, central.txt
    - central copy nodes
    - central init
    - central query statistics.py
    - central scheduler.py
    - folder network script: automate droplet.py, droplet generate random files, droplet init, droplet scp time transfer
    - folder mongo control
        - mongo scrip/
          - server.py
          - install package.py
        - mongo control/
          - insert to mongo.py
          - read info.py
          - read info.pyc
          - install package.py
          - jobs.py
          - ip path

Note that while we currently use an implementation of HEFT for use with CIRCE, other schedulers may be used as well.

# References
[1] H. Topcuoglu, S. Hariri, M.Y. Wu, Performance-Effective and Low-Complexity Task
Scheduling for Heterogeneous Computing, IEEE Transactions on Parallel and
Distributed Systems, Vol. 13, No. 3, pp. 260 - 274, 2002.

[2] Ouyang Liduo, HEFT Implementation Original Source Code, https://github.com/oyld/heft  (we have modified this code in ours.)

# Acknowledgement
This material is based upon work supported by Defense Advanced Research Projects Agency (DARPA) under Contract No. HR001117C0053. Any views, opinions, and/or findings expressed are those of the author(s) and should not be interpreted as representing the official views or policies of the Department of Defense or the U.S. Government.

