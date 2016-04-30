# CS5300 Project 2
** Fast Convergence PageRank in MapReduce **

**_Samir Borle (smb435), Kunhe Chen (kc853), Richard Quan (rq32)_**

### Introduction
Over code contains five `.java` files in `./src/project1b` directory. A makefile is included for easy compilation using `javac` and `jar` from linux-based computers (tested on Mac only). For successful compilation using the default setup, required hadoop library .jar files are included in `./hadoop_lib` directory.

### Java files
There are five `.java` files included in our final submission
1. `fileFilter.java`: filter the file according to netID.
    * We prefiltered the edge.txt file according to the netID (**kc853**). According to the project description, we used the parameters:
    ```
    double fromNetID = 0.358; // From kc853
	double rejectMin = 0.9 * fromNetID;
	double rejectLimit = rejectMin + 0.01;
    ```
    and then reject edges associated with values `rd`
    ```
    rejectMin < rd < rejectLimit
    ```
    * The program reads from `edges.txt` file and output to `filtered_edges.txt` file, which we will use for our project. The output format is
    ```
    u    v   deg(u)  pr(u)
    ```
    where `deg(u)` counts the number of outlinks from node `u` and `pr(u)` is defaulted to `1/N = 1/685229`
    * This file is not needed for running because it has been done locally and `filtered_edges.txt` is available on S3. See later sections for more details.
2. `PageRank.java`: compute the simple node-to-node pagerank update using the hadoop MapReduce. The program consists of three classes - a mapper class, a reducer class and a driver(main) class.
    * The mapper class reads a line from our `filtered_edges.txt` file, parse the line and uses nodeID as keys to map the values to reducers. For each value(line) the mapper reads, it sends two sets of lists to reducers
    ```
    <v, List<u, v, deg(u), pr(u)>>
    <u, List<u, v, deg(u), pr(u)>>
    ```
    The first list is used to update pagerank values for each node `v`. The second list is used after the pagerank update and each node `u(w)` will be able to emit its results (see reducer class description).
    * The reducer class reads a key and a list of values associated with the key, namely
    ```
    <key, List<key, v, deg(u), pr(u)>>
    ```
    Then it parses the entries in the list and put them into two sets `influx` and `efflux` by whether the source node `u` matches the key. Then the reducer updates the pagerank `pr_new(v)` for node `u` to node `v(key)` and computes the node residual according to this new pagerank. When the computation is done, the reducer emits its result using the information contained in the `efflux` set (`<w, List<v, w, deg(v), pr(v)>>`) with the format as
    ```
    v    w   deg(v)  pr_new(v)
    ```
    The reducer also increment the counter according to the node residual it computed from earlier.
    * The driver class (main function) deals with job configuration and system output (for residual after each iteration/pass). The program terminates after iterating `MAXIT (= 5)` times or when the residual is smaller than `TOLERANCE = 0.0001`.
2. `blockedPageRank.java`: computes the pagerank by blocks. The program is similar to `PageRank.java` in structure with changes in mapper and reducer classes.
    * The mapper class uses blockIDs as keys instead of nodeIDs. The blockIDs are computed from nodeIDs using `getBlockID(nodeID)` and a list of boundary values from blocking (obtained from file `block.txt`). Besides that, it is similar to the mapper class in node-to-node pagerank update.
    * The reducer class organizes the input values with three sets - a set of all nodes in the block (`nodes`), a set of edges coming into the block (`influx`) and a set of edges leaving the block (`efflux`). Then the reducer does the following (Gauss-Seidel) update:
    ```
    for each v in nodes:
        for each u in {u | u -> v} (a subset of influx)
            update pr(v) according to u;
        update pr(v) for each v in influx;
        add to block residual;
    ```
    The reducer will go through this process until the average residual converges, and it will computes the block residual. Then the reducer emits the results according to the `efflux` set, as well as block residual value and total iteration number.
    * The driver class is similar to the one in `PageRank.java`. It terminates when the average residual block is small enough.
2. `blockedJacobiPageRank.java`: does the same job as `blcokedPageRank.java` with a Jacobi method.
    * The reducer does not update the `pr(v)` in the `influx` set until the pagerank update for each node `v` in `nodes` is complete.
3. `randomPageRank.java`: does a similar job as `blockedPageRank.java` but using a hash function to generate key instead of using the given block partition.
    * The map computes the key by calling `hash32shiftmult(nodeID)` and distributes the information to reducers.
    * The reducer examines each edge in the input list and see if the edge belongs to the `influx` or(and) the `efflux` set, as well as compiling a `nodes` set from the list.

### Results

Results for simple pagerank:
```
====== After 1 iterations, the residual is2.32679402 ======
====== After 2 iterations, the residual is0.32678479 ======
====== After 3 iterations, the residual is0.19788537 ======
====== After 4 iterations, the residual is0.10093302 ======
====== After 5 iterations, the residual is0.07000755 ======
```
