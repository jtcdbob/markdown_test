# CS5300 Problems 3

**Kunhe Chen (kc853)**

### Question 1

In order to include DB0(legacy) in the 2PC-based distributed system, we want to modify the protocol so that the coordinator will have a way to probe whether DB0 is prepared to commit.

The protocol excutes the following steps:

1. Send `prepare` to DB1-DBN. Wait for responses.
2. If any DB replied with a `NO` or timed out, send `abort` to all DB1-DBN. Otherwise, if all the database servers are ready to commit (N `YES` received by the coordinator), proceed to check DB0.
3. For DB0, coordinator will have it add the commited transaction to a table (unique to DB0 because it's a legacy DB) by a commit. Then the coordinator will try to read the latest commited transaction from DB0. If the records are not consistent, DB0 is not ready to commit, so send `abort` to all DB1-DBN. Otherwise, DB0 is prepared and can commit T.
4. The coordinator will have DB0-DBN commit the transaction T.

In summary, the modified protocol treats DB1-DBN as the 2PC-capable database servers are they are, while using a record table in DB0 to make up for the lack of prepared state. If coordinator can read the latest transaction from DB0 after sending the commit request, it indicates that DB0 is alive and ready for commit (essentially a `YES` from DB0). The rest of this protocol is largely unchanged from the original, since DB0 can participate in the voting procedure now.

### Question 2

#### Part a:

In unit time, the total number of transactions (txn) is

```
Txn = TPS * Acts * ActTime
```

Assuming that acts are half-way through on average, the total number of objects that are locked is

```
n = Txn * Acts / 2 = TPS * ActTime * Acts^2 / 2
```

So the probablity to find an object locked is
```
P = n/DB = TPS \cdot ActTime \cdot Acts^2/ (2DB)
```

Assume that P << 1, the probability to wait (for at least one lock) is

```
PW = 1 - (1-P)^Acts ~ P\cdot Acts = TPS * ActTime * Acts^3/ (2*DB)
```

Suppose Acts=1

```
PW = TPS * ActTime/ (2*DB)
```

#### Part b:

To have a deadlock, more than one locks happen at the same time for a transaction

$$
PD = \sum_{k=2}^\infty PW^k / Txn
$$

Since PW is small, the summation is dominated by the first term where k=2:

```
PD = PW^2 / Txn = TPS * ActTime * Acts^5/ (4*DB^2)
```

Suppose Acts=1

```
PD = TPS * ActTime / (4*DB^2)
```

#### Part c:

For each (red/blue-only) transaction, the PW is

```
PW_i = TPS * ActTime/ (2*DB_i)
```

Consider a 50-50 split of the total transactions, we can simply average the probability:

$$
PW = \frac{PW_r + PW_b}{2} = \frac{1}{2}\left(\frac{TPS * ActTime}{2\alpha DB} + \frac{TPS * ActTime}{2\beta DB}\right) = \frac{TPS * ActTime}{4DB} \left(\frac{1}{\alpha} + \frac{1}{\beta} \right)
$$

#### Part d:

Now Acts=2 and transactions are "RB-ordered", the PW for red will have 2*ActTime instead because the lock will not release until the acts on both red and blue are complete. And

$$
PW = 1 - (1-P_r)^Acts(1-P_b)^Acts \approx Acts(P_r + P_b)
$$

Acts=1 for each transaction type, so

$$
PW = \frac{TPS * {\bf 2}ActTime}{2\alpha DB} + \frac{TPS * ActTime}{2\alpha DB} = \frac{TPS * ActTime}{2DB} \left(\frac{2}{\alpha} + \frac{1}{\beta} \right)
$$

#### Part e:

Because Acts=2 for each transaction,

$$
PD = \frac{PW^2}{Txn} = \left(\frac{TPS * ActTime}{2DB}\right)^2 \left(\frac{2}{\alpha} + \frac{1}{\beta} \right)^2 \frac{1}{TPS* Acts *ActTime}
$$

$$
\Rightarrow PD = \left(\frac{TPS * ActTime}{4DB}\right) \left(\frac{2}{\alpha} + \frac{1}{\beta} \right)^2
$$

#### Part f:

If it is "BR-ordered" instead, we can simply switch the constant $\alpha$ and $\beta$:

$$
PD = \left(\frac{TPS * ActTime}{4DB}\right) \left(\frac{2}{\beta} + \frac{1}{\alpha} \right)^2
$$

#### Part g:

If there are more blue items than red items, $\alpha < \beta$, we know that

$$
\frac{2}{\alpha} + \frac{1}{\beta} > \frac{2}{\beta} + \frac{1}{\alpha}
$$

"RB-ordered" transactions will have a larger PW and PD. Therefore, we would recommend using "BR-ordered" transcations.

### Question 3

#### Part a:

The algorithm, described in the Pregel paper, terminates the pagerank update when every node reaches its fixed point. In this case, the termination condition will be when every node is at its fixed point, i.e., `newrv = oldrv` for each node.

To implement this termination test, we can use Hadoop counters to keep track of the total number of stablized nodes. In each iteration, if a node is at its fixed point, it increment the counter by 1. In the driver function, the program will terminate when `counters==N`, where `N` is the total number of nodes in the system. A more relaxed termination condition can be `counters > b * N`, where `0 <= b <= 1` relaxes the termination condition.

#### Part b:

Suppose the graph is partitioned into blocks, we can take advantage of it by localizing the computation in each block. The MapReduce computation should be modified as following:

* The Mapper function no longer uses the nodeID `v` as the key but uses the blockID that is associated with the node.
* The reducer will get information about nodes in its block, as well as nodes that are outside the block but flow into the block. The reducer will treat the outside nodes as constant and carry out local computation till the pagerank update converges. The local update will implement the algorithm in the Pregel paper and terminates when all (or enough) nodes have reached fixed points.
* The reducer also reports the number of nodes `M` that are in fixed point before the local pagerank update starts.
* The driver function will terminate the computation when `M==N` (or its relaxed counterpart).

This way, the MapReduce computation will take advantage of the locality of the graph. It distributes global information and carries out computation on each reducer till the pagerank converges locally. The local convergence information at the beginning of computation is collected to measure whether the pagerank has converged globally.
