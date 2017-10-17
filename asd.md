Reactive query optimization
==

Employ runtime collected informations to improve execution reliability.

Goals:

1. re-execute the query with a different plan.
2. oversee query execution and collect runtime statistics

Pieces:

 * Planning can adapt to use cases if it gets relatively usable estimations
 * `Tez` provides a way to collect statistics on all the vertices
	 * some stats are already available
	 * it can be extended to publish other counters

I'm thinking in the following: since this seems like a place where decisions like re-planing or not the query could be made; I position it above the Driver.

So far I’ve see the most viable is the following approach:

* Introduce a new abstraction layer: the goal is to create a communication boundary between the ‘ql’ an the ‘service’ module via this interface. 
	* I would call it something like: `QueryExecutor`? 
	* Add a default implementation and enable HS2 to use this interface
* Make use of the new interface - add a sample `QueryExecutor`
	* if the query fails; re-execute while forcing the all joins to be ShuffleJoins
	* skip query re-execution in case the plan is the same.
* Enable the the `QueryExecutor` to tap into the query execution more deeply
	* use existing hook points to install the `QE` listeners 
* Generalize statistics codes to accept a `StatisticsSource` ; change existing codes to use a `MetaStoreStatisticsSource`
* Add `RuntimeStatisticsSource`
* Collect RuntimeStatistics
	* from a failed plan; and use it during replanning
	* store stats in a temporal cache
		* prune runtime stats in case table changes
			* this can be relaxed later; but to be on the safe side...
		* use cacheKey `hash(query,operator.name)`
			* this could aid exact queries
		* use cacheKey `hash(ancestorsOf(operator))`
			* this could enable the reuse of statistics for the same subtree
* Possibly store RuntimeStatistics for a longer period
	* this will possibly need changes at the metastore side also.
	* a simple KV storage would be more than sufficient

