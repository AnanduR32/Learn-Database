# Cassandra

- Peer-to-peer distributed system wherein data is replicated among multiple nodes
- Each node is equal and where specific data is stored.
- Nodes are logically arranged in continuous loop called a ring
- Data is spread across the ring using consistent hashing
- This ring of nodes make up the datacenters, which inturn makes up a cluster
- to track health of cluster, nodes communicate with each other every second using peer-to-peer communication protocol called '*Gossip*'
- *Snitches* determine the physical topology of network and thus routing requests and safely distribute data replica across different physical racks to withstand power failures.

Usecases:

- Event logging
- Content management
- Counters - using CounterColumnType, to count and categorize visitors
- Expiring usage - Use expiring columns with TTL
