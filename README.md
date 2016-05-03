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
3.
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

4. Notes on running the files

    * Make sure the file path is correctly configured to the S3 file before compiling.

    * Make sure to use the java with matching version to the one hadoop accepts.

### Results

Results for simple pagerank:

```
====== After 1 iterations, the residual is 2.32679402 ======
====== After 2 iterations, the residual is 0.32678479 ======
====== After 3 iterations, the residual is 0.19788537 ======
====== After 4 iterations, the residual is 0.10093302 ======
====== After 5 iterations, the residual is 0.07000755 ======
```
The total time for node-to-node PageRank update to get residual of 0.07 is 4 minutes.

Results for blocked pagerank (Gauss-Seidel):

```
**********   Defaulted to Gauss-Seidel   **********
====== The 1 pass takes average of 1007.0 iterations to complete ======
====== After 1 pass, the residual is 0.004377 ======
====== The 2 pass takes average of 599.0 iterations to complete ======
====== After 2 pass, the residual is 0.02737 ======
====== The 3 pass takes average of 548.0 iterations to complete ======
====== After 3 pass, the residual is 0.023273 ======
====== The 4 pass takes average of 442.0 iterations to complete ======
====== After 4 pass, the residual is 0.010286 ======
====== The 5 pass takes average of 336.0 iterations to complete ======
====== After 5 pass, the residual is 0.004835 ======
====== The 6 pass takes average of 245.0 iterations to complete ======
====== After 6 pass, the residual is 0.001905 ======
====== The 7 pass takes average of 189.0 iterations to complete ======
====== After 7 pass, the residual is 9.34E-4 ======
====== The 8 pass takes average of 141.0 iterations to complete ======
====== After 8 pass, the residual is 4.24E-4 ======
====== The 9 pass takes average of 112.0 iterations to complete ======
====== After 9 pass, the residual is 2.29E-4 ======
====== The 10 pass takes average of 86.0 iterations to complete ======
====== After 10 pass, the residual is 1.13E-4 ======
====== The 11 pass takes average of 79.0 iterations to complete ======
====== After 11 pass, the residual is 5.3E-5 ======
!! MAPREDUCE ITERATION CONVERGES AFTER 11 PASS
```
The total time for Gauss-Seidel to converge is 28 minutes.


Results for blocked pagerank (Jacobi):

```
******************   Jacobi   ******************
====== The 1 pass takes average of 1858.0 iterations to complete ======
====== After 1 pass, the residual is 0.004319 ======
====== The 2 pass takes average of 908.0 iterations to complete ======
====== After 2 pass, the residual is 0.033498 ======
====== The 3 pass takes average of 822.0 iterations to complete ======
====== After 3 pass, the residual is 0.023108 ======
====== The 4 pass takes average of 610.0 iterations to complete ======
====== After 4 pass, the residual is 0.010044 ======
====== The 5 pass takes average of 448.0 iterations to complete ======
====== After 5 pass, the residual is 0.004621 ======
====== The 6 pass takes average of 303.0 iterations to complete ======
====== After 6 pass, the residual is 0.001752 ======
====== The 7 pass takes average of 223.0 iterations to complete ======
====== After 7 pass, the residual is 7.98E-4 ======
====== The 8 pass takes average of 135.0 iterations to complete ======
====== After 8 pass, the residual is 3.05E-4 ======
====== The 9 pass takes average of 112.0 iterations to complete ======
====== After 9 pass, the residual is 1.38E-4 ======
====== The 10 pass takes average of 88.0 iterations to complete ======
====== After 10 pass, the residual is 8.4E-5 ======
!! MAPREDUCE ITERATION CONVERGES AFTER 10 PASS
```

The total time for Jacobi to converge is 34 minutes. Jacobi takes fewer passes to converge to smaller residual, however the (real-life) time for it to converge is longer than Gauss-Seidel with the same cluster specification. Both (Jacobi and GS) are faster than the simple one, in the sense that 1 iteration (average time 3.4 minutes for Jacobi and 2.5 minutes for GS) is sufficient to bring the residual to ~0.004 < 0.07 from simple version (around 4 minutes).


Results for randomized pagerank:

```
**********   Defaulted to Gauss-Seidel   **********
====== The 1 pass takes average of 236.0 iterations to complete ======
====== After 1 pass, the residual is 0.039965 ======
====== The 2 pass takes average of 210.0 iterations to complete ======
====== After 2 pass, the residual is 0.14347 ======
====== The 3 pass takes average of 204.0 iterations to complete ======
====== After 3 pass, the residual is 0.175513 ======
====== The 4 pass takes average of 204.0 iterations to complete ======
====== After 4 pass, the residual is 0.086775 ======
====== The 5 pass takes average of 204.0 iterations to complete ======
====== After 5 pass, the residual is 0.058101 ======
====== The 6 pass takes average of 204.0 iterations to complete ======
====== After 6 pass, the residual is 0.032699 ======
```

The randomized pagerank update is to show that it has much slower convergence rate than one with a "good" blocking.

### Sample Result for blocked PageRank update

Gauss-Seidel:

|     0     |     1     |   10328   |   20373   |   20374   |   20375   |
|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|
| 1.5038E-5 | 5.1898E-5 | 4.3738E-7 | 2.1890E-7 | 2.1890E-7 | 2.1890E-7 |
|   30629   |   30630   |   40645   |   50462   |   50463   |   50464   |
| 2.1890E-7 | 2.1890E-7 | 2.1890E-7 | 9.6610E-4 |   0.0044  | 2.0334E-5 |
|   60841   |   60843   |   70591   |   70592   |   80119   |   80120   |
| 2.9738E-6 | 2.0648E-5 | 2.1890E-7 | 2.0332E-5 | 9.0100E-7 | 7.4252E-7 |
|   90497   |   90498   |   100501  |   100502  |   110567  |   110568  |
| 9.3633E-7 | 1.1325E-5 | 1.0181E-6 | 9.4769E-7 | 3.3816E-4 | 8.0271E-5 |

Complete file can be found online in:

[s3://edu-cornell-cs5300-project2-kc853/gs_data.zip](https://s3-us-west-2.amazonaws.com/edu-cornell-cs5300-project2-kc853/gs_data.zip) (Gauss-Seidel after 11 iterations)
and

[s3://edu-cornell-cs5300-project2-kc853/gs_data.zip](https://s3-us-west-2.amazonaws.com/edu-cornell-cs5300-project2-kc853/jacobi_data.zip) (Jacobi after 10 iterations)

[url=https://flic.kr/p/GpLVby][img]https://farm8.staticflickr.com/7370/26523313960_79aa29bc31_z.jpg[/img]

