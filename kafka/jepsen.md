## Jepsen: Kafka

This is my gloss of Kyle Kingsbury's (aka @aphyr) article about identifying a failure mode in Apache Kafka. That article can be found [here](https://aphyr.com/posts/293-jepsen-kafka).

I skipped some of the core details about Kafka that are laid out in this post. There are other sections of these notes that provide more info about those topics. 

This is article is from 2013, so I have very little sense of what from this is still applicable.

#### Core claim

In version 0.8.0.0, Kafka claims that they are able to survive the loss of F-1 nodes in an F node cluster without sacrificing consistency. This would give them something like CP, with pretty good A as well. This is an improvement over typical system that can only survive `F/2 - 1` failures. 

Kingsbury shows that this is not true.

#### Design

Consider a 3 node cluster, with one leader (Node 1) and two followers. At the start the size of the In Sync Replica set (ISR), as managed by ZooKeeper, is 3. 

But then the leader is partitioned from the two followers by some iptables shenanigans. During this time, the leader continues to accept writes to the system. But then the leader loses its claim in ZooKeeper to be the leader, and will continue to accept writes to the system. 

Then, the followers come back, and Node 2 is the leader. At this point, all the writes that were made to Node 1 during the partition were lost. 

#### Conclusion

It seems like Kafka probably can't accept up to F-1 failures. However, Jay Kreps follows up with [an article](http://blog.empathybox.com/post/62279088548/a-few-notes-on-kafka-and-jepsen) considering the operational constraints of LinkedIn.

The choice here, by Kreps:

```
Is it better to be alive and wrong or right and dead?
```

If the cluster goes down to one node, that node could just spit out a "sorry we're not home right now" message. But then this is downtime, and is not strictly necessary downtime, given that the leader node is still up. During this "emergency situation", if leader can hold on to its leadership claim until:
1. another node rejoins the cluster.
2. that node copies all of the "emergency" writes

then availability and consistency were never sacrificed. But they also got lucky.

