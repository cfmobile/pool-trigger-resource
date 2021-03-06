# pool-trigger-resource
A resource solely for triggering concourse builds when resources are
added to pools.

When a single job is configured to trigger on this resource it
guarantees one triggered build for every unclaimed lock that is in the
pool when the resource is created and one for each lock added
afterwards.

Do not use it to get or put locks.

### ATTENTION
##### Behaviour is only guaranteed if 
###### 1. A SINGLE job triggers on the configured pool.
###### 2. No other job claims locks on the same pool.


#### Usage

Configure a `pool-trigger` resource in addition to a normal `pool-resource` 
resource for the same pool with identical parameters. Then add a `get`
step to your job referencing the `pool-trigger` and a `put` referencing
the `pool-resource` with `acquire: true`. Don't expect anything useful
from the `pool-trigger` as an input - the `in` script is a no-op.

An example showing an infinite loop that moves locks back and forth
between two pools forever can be found here: `./examples/red<>blue.yml`



#### Behaviour

##### In
no-op

##### Out
no-op

##### Check
1. Checks for newly added locks since the last version (or any currently
   unclaimed if starting fresh with no previous version) and adds these
   to the tally in `.pending-triggers`
2. Checks for removed locks and subtracts them from the tally in `.pending-removals`
3. Any locks removed in excess of `.pending-removals` are removed from `.pending-triggers`
5. If the `.pending-triggers` tally is greater than zero it decrements
   the `.pending-triggers` and increments `.pending-removals` and returns the previous commit hash reference
6. If the `.pending-triggers` tally is zero it returns no new references

##### Failures
1. If `check` fails to push the updated `.pending-triggers` tally it will 
   fail out. This should only happen if other commits are pushed in 
   between fetching and pushing the tally. It should resolve itself on 
   the next check with no intervention.
2. If a build fails before the claim step happens (usually due to a concourse/aws connection error) it will be necessary to manually re-trigger the build.
3. If for any reason things get all wonky things can be re-initialized by deleting `.pending-triggers` or `.pending-removals`.
