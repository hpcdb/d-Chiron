# d-Chiron

## Dependecies
- Java JRE 1.7 64 bit or newer
- MySQL Cluster
- libaio1 64 bit
- The front node (node from which the other processes will start) should be able to access the other nodes via ssh using public key.

## Setup
1. Clone repository:

    ```bash
    $ git clone https://github.com/vssousa/d-chiron.git
    ```
2. Download MySQL Cluster:
    
    [Linux Generic (glibc 2.5) (x86, 64-bit), Compressed TAR Archive](http://dev.mysql.com/downloads/cluster/)
3. Unpack MySQL Cluster:

    ```bash
    $ tar -xvvf mysql-cluster-gpl-X.Y.Z-plataform.tar.gz
    ```
4. Change working directory to d-Chiron:
    
    ```bash
    $ cd d-chiron/d-chiron
    ```
5.  Setup  MySQL Cluster for d-Chiron:
    
    ```bash
    $ ./setupmysql.sh <mysql_path> <mysql_directory> <data_directory> <number_of_nodes>
    ```
 - `mysql_path`: fullpath to the unpacked MySQL Cluster directory
 - `mysql_directory`: root directory where MySQL cluster directories will be placed into
 - `data_directory`: base directory where data from data nodes will be placed into
 - `number_of_nodes`: Maximum number of nodes that will run MySQL Cluster. If there are, for example, 5 database nodes (1 mgm_node, 2 sql_nodes, 2 data_nodes), this number should be at least 5.

## Configuration

- Edit dbcluster/installation.properties:

    ```bash
    $ vi dbcluster/installation.properties
    ```
    - [Follow this link for an example](dbcluster/installation.properties.example)
    - Define installationRootDirectory and base_data_dir as you defined in Setup#5 for mysql_root_directory an base_data_directory, respectively.
    - Chose the number of nodes used by  MySQL Cluster
        - number_of_datanodes
        - number_of_sqlnodes
        - number_of_mgm_node
    - All other configurations may be left as the example file.

## Workflow Configuration

We will use the [Risers Fatigue Analysis workflow](../rfa-synthetic) as example.

1. Change working directory to `rfa-synthetic/rfa-dchiron`:

    ```sh
    $ cd ../rfa-synthetic/rfa-dchiron
    ```
2. [Use the XML file from RFA workflow](../rfa-synthetic/rfa-dchiron/rfa-dchiron-wf.xml):
    
    ```sh
    $ vi rfa-dchiron-wf.xml
    ```
    - Replace `full/to/path/repositoryroot/` occurrences with the full path to this repository root directory
    
3. Edit the [machines.conf](../rfa-synthetic/rfa-dchiron/machines.conf) file with nodes hostnames, ports, and ranks for MPI initialization:
    
    ```sh
    $ vi machines.conf
    ```
   Example:
    ```
    # Number of processes
    2
    # Protocol switch limit
    131072
    #Entry in the form of machinename@port@rank
    node1-hostname@20348@0
    node2-hostname@20348@1
    ```

4. Edit the [input.dataset](../rfa-synthetic/rfa-dchiron/exp/input.dataset) file to adjust the amount of data and parallel tasks your workflow will process. This is a CSV file. Each row in this dataset represents one subgraph of task executions of an RFA run.
The meaning of each field is as follows.

    - `ID`: identifier of the subgraph
    - `SPLITMAP`: factor that controls the average cost of each *Data Gathering* task execution.
    - `SPLITFACTOR`: factor that controls the amount of parallel tasks that will be generated (at the *Data Gathering* activity).
    - `MAP1` and `MAP2`: factors that control the average cost of each *Preprocessing* and *Stress Analysis* task execution.
    - `FILTER1` and `FILTER2`: factors that control the average cost of each *Stress Critical Case Selection* and *Curvature Critical Case Selection* task execution, respectively.
    - `F1` and `F2`: factors that control the retention of tuples (in the dataflow) in the *Stress Critical Case Selection* and *Curvature Critical Case Selection* activities, respectively.
    - `REDUCE`: factor that controls the average cost of each *Compress Results* task execution.
    - `REDUCEFACTOR`: factor that controls the amount of resulting tuples after going through the reduce (*Compress Results*) activity.
    <br/>


    For the theoretical reference behind the choices for these factors, please see [this reference](https://hal-lirmm.ccsd.cnrs.fr/lirmm-00749968/document):
    ```
    Fernando Chirigati, Vitor Silva, Eduardo Ogasawara, Jonas Dias, Fabio Porto, et al. Evaluating Parameter Sweep Workflows in High Performance Computing. SWEET'12: International Workshop on Scalable Workflow Enactment Engines and Technologies, 2012, Scottsdale, Arizona, United States. pp.10.
    ```

    The source code for the Risers Fatigue Analysis workflow tasks generator is available [here](../rfa-synthetic/rfa-activities).

## Run

Still using the [Risers Fatigue Analysis workflow in d-Chiron](../rfa-synthetic/rfa-dchiron/) as example, follow the next commands to run the workflow. The scripts used in this section are [here](../rfa-synthetic/rfa-dchiron).

1. Start MySQL Cluster instances:

    ```bash
    $ ./invokestartdb.sh <mysql_root_directory>
    ```

2. Submit workflow execution:

    ```bash
    $ ./invokewf.sh <mysql_root_directory>
    ```
3. Shutdown MySQL Cluster instances:

    ```bash
    $ ./invokeshutdown.sh <mysql_root_directory>
    ```

## Monitor

Optionally, you can monitor the execution by specifying periodic queries in the workflow database.

1. Add new monitoring queries before or during workflow execution, informing the monitoring interval. Example:

    ```sql
    insert into dchiron.umonitoring(monitoringinterval, monitoringquery) values (
        10, 
        'select avg(g.map2) from dchiron.eactivation artask, rfa.ogatherdatafromrisers g, rfa.opreprocessing p, rfa.oanalyzerisers r where r.previoustaskid = p.nexttaskid and p.previoustaskid = g.nexttaskid and r.previoustaskid = artask.taskid and (artask.endtime-artask.starttime) > (select avg(endtime-starttime) from scc2.eactivation where actid=3);'
    );
    ```

1. Start the `Monitor` module:

    ```bash
    $ java -jar Monitor.jar --start
    ```
	
1. For each monitoring query registered in the `UMonitoring` table, the result set is stored in `UMonitoringResult` table during workflow execution at each time interval defined for the query. 
	
## Steer

If you need, d-Chiron allows you to steer the execution by adapting the workflow data at runtime.

#### Removing slices

1. First, determine the select predicates to determine a slice to be removed. For example, `map1 > 70000`. 

2. Then, run a steering query to cut off the slice:

    ```bash
    $ java -jar Steer.jar --xml=rfa-dchiron-wf.xml --user=peter --type=REMOVE_TUPLES --tasksquery="select  nexttaskid  from wf1.ogatherdatafromrisers where map1 > 70000" 
    ````
