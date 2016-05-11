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

In unit time, the total number of transactions (txn) is

$$$
Txn = TPS \cdot Acts \cdot ActTime
$$$

Assuming that acts are half-way through on average, the total number of objects that are locked is

$$$
n = Txn \cdot Acts / 2 = TPS \cdot ActTime \cdot Acts^2/2
$$$

So the probablity to find an object locked is
$$$
P = n/DB = TPS \cdot ActTime \cdot Acts^2/ (2DB)
$$$

Assume that $P\ll 1$, the probability to wait (for at least one lock) is
$$$
PW = 1 - (1-P)^Acts \approx P\cdot Acts = TPS \cdot ActTime \cdot Acts^3/ (2DB)
$$$

### Question 3
