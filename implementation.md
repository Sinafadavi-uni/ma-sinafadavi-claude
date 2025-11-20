\section{Architectural plan of datastore}

Below is a complete architecture for the datastore system. It is based only on three key strategies:

\begin{itemize}
    \item Storing multiple copies of data for safety,
    \item Using a decentralized method to find where data is stored,
    \item And repairing differences between replicas quickly and efficiently using Merkle trees.
\end{itemize}
This design ensures reliability, scalability, and low network usage, even when nodes fail or go offline.

The datastore system is a decentralized key–value store that replicates data across multiple nodes. It has no single point of failure because each piece of data is stored on more than one node. Keys are assigned to nodes using consistent hashing, so there is no need for a central coordinator. When replicas become inconsistent, the system uses Merkle trees to find and fix differences efficiently.

\subsection{Core strategies integrated}

\begin{itemize}
    \item \textbf{Multiple redundant copies (no single point of failure)}
\end{itemize}

Each key–value pair is stored on more than one node. The system uses a replication factor of at least two (\(r \geq 2\)), ensuring that multiple copies are always maintained.



When a write happens, all replicas are updated at the same time. The write only succeeds if all replicas confirm it.
This means if one node fails or goes down, the data is still available on the other nodes. There is no single point of failure.

\begin{itemize}
    \item \textbf{Decentralized data mapping (no central coordinator)}
\end{itemize}

The nodes are organized in a ring using consistent hashing.
Any node or client can figure out which nodes are responsible for a given key: it just hashes the key and looks at the ring to find the correct place by moving clockwise.
Because of this, there’s no need for a central directory or metadata server. Nodes can join or leave the system without requiring changes across the whole cluster.

\begin{itemize}
    \item \textbf{Efficient incremental repair using Merkle trees}
\end{itemize}

Each node builds a Merkle tree that represents its stored data. This tree acts like a summary.
Replicas regularly exchange their root hashes. If the roots are different, they go deeper into the tree to find which parts don’t match. Then, only the keys and values from those mismatched sections are transferred.
This process, called anti-entropy, keeps all replicas in sync with very little data sent over the network. It makes repairs fast and efficient.



\begingroup
  \setlength{\parskip}{1em}   % or some length you like
  \setlength{\parindent}{0pt} % optional: remove indentation in this block








\section{Design overview}

\subsection{Decentralized data mapping }
Lays the routing substrate every other replication feature depends on; without it, we don’t know where replicas belong or how to reach them.

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}
The system’s main job is to decide where data should be stored which node gets which key without needing a central coordinator. It does this in a predictable and reliable way using consistent hashing.


\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{membership\_manager:} This part keeps track of which nodes are currently active, whether they’re healthy, and how virtual nodes (tokens) are assigned across the ring.

\textbf{hash\_ring:} This is the implementation of consistent hashing. When given a key, it produces an ordered list of nodes that should store it.

\textbf{routing\_client:} A small library used by brokers, executors, and datastores. It lets any node find out which replicas are responsible for a given key by using the current hash ring.

\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}

When a node joins or leaves the network, it sends a message about this change. The membership manager picks up this event and updates the hash ring accordingly.
Other nodes eventually receive this updated ring state. Then, whenever a client needs to read or write a key, it uses its local copy of the ring to compute the correct set of target nodes all locally, with no need to ask a central server.
This means reads and writes go directly to the right place, fast and without a central directory.


\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

- The function f(k) takes a key k and returns an ordered list of N nodes that are responsible for storing it.

- Hash ring snapshots include version information so different nodes can detect outdated views and gradually converge to the same state over time.


\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}


First, we define the data structures for the hash ring. This includes node tokens (positions on the ring), support for virtual nodes to balance load better, and version numbers for the ring itself so changes can be tracked. We also decide how this data is saved and shared between nodes using a simple format like JSON or MessagePack.

Next, we implement a gossip-based membership system where nodes send heartbeat messages to share their status. The membership manager collects this information and uses it to build a list of live nodes. Based on that list, it assigns tokens in a consistent and predictable way, so every node calculates the same assignments given the same set of active nodes.

We then write a pure function one that always gives the same output for the same input that takes a key and returns the list of nodes responsible for storing it. This function is used by all components. We also add helper methods to find the next nodes in the ring, which helps during replication when writing data to multiple replicas.

The routing client is updated to use this local ring logic. Brokers, executors, and datastores now compute where data should go on their own, without asking a central server. This replaces any old lookup mechanism that depended on a coordinator.


\textbf{To help monitor the system, we add observability features:}

- A ring checksum to quickly check if nodes have similar views

- Metrics that show how often keys are remapped when nodes join or leave

- Warnings if different nodes see conflicting ring versions

\subsection{Multiple redundant copies }
Builds on the mapping layer to guarantee availability by placing and tracking replicas across the ring.

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}
The system makes sure that every piece of data (each key) is stored on at least N different nodes, so it won’t be lost if one node fails. But it also respects network limits especially important in low-bandwidth environments by not copying too much or too often.

\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{replication\_controller:} This runs inside brokers or datastores and manages how data is copied. It decides which nodes should store a given key, sends out the write requests, and keeps track of which replicas have responded.

\textbf{replica\_metadata\_store:} A small database (possibly kept at the broker) that stores information about each key’s replicas  like their current state, version numbers, and how many responses are needed for success (quorum).

\textbf{consistency\_policy:} A flexible layer that defines how strong consistency should be. For example, it sets values like W (how many replicas must confirm a write) and R (how many to read from), and supports features like 
 to handle temporarily offline nodes.

\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}
When a client sends a write request, it goes to a coordinator node (like a broker or a designated datastore). The replication controller looks up the key in the hash ring and gets a list of N preferred nodes (the preference list). It sends the write to all of them, but only waits for W acknowledgments before saying the write succeeded.
Once confirmed, it updates the replica metadata. If some replicas were unreachable, their pending writes are added to a hinted handoff queue and sent later when they come back.

\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

A write is only considered durable once it receives the required number of successful replies (based on W).
Each replica has a state such as fresh, syncing, stale, or tombstoned that helps the system know what needs repair later.
For executor results, the “first valid result wins” rule still applies. But now, the system uses replica metadata to check if a result comes from an up-to-date copy before accepting it.

\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}


We start by defining the replication settings, like the default number of copies (N), how many confirmations are needed for a write to succeed (W), and how many replicas to read from (R). These values can be changed later, so we make them part of a config service that’s easy to update.

Next, we extend the datastore interface with a put function that takes not just a key and value, but also a version and an origin ID. This helps track updates using version vectors, so we can handle concurrent changes. The operation is designed to be idempotent meaning doing it more than once doesn’t cause problems which is important when retries happen.


In brokers and datastores, we build the write pipeline:

First, we use the hash ring to find the list of N target nodes (the preference list).


Then we send the write to all of them at the same time.
Wait only for W successful replies before marking the write as complete.
If some nodes don’t respond, their pending writes go into a hinted handoff queue for later delivery.
We also add a background process a reconciler  that checks this queue regularly. When a previously offline node comes back online, the system tries to deliver the missed writes.

For reading, we update the read path so clients don’t just pick any replica. Instead, they check the replica metadata and choose the one with the most up-to-date version. This sets the stage for read repair fixing inconsistencies when they’re found.

Finally, we write tests to make sure everything works under real-world conditions.

\textbf{These include:}

- Simulating node failures

- Checking that quorum rules are respected

- Testing if hinted handoff correctly replays missed writes

- We also run chaos-style scenarios based on the design doc like killing a broker mid-write or creating network partitions  to see how well the system recovers

\subsection{Efficient incremental repair with Merkle trees }

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}


Once data is replicated across nodes, the system must make sure all copies stay in sync over time. The goal is to detect when replicas have become different (due to failures or network issues) and fix them but without sending large amounts of data. Only the missing or outdated parts should be transferred.

\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{merkle\_tree\_builder:} This runs on each node and builds a Merkle tree for its stored data. The tree covers key ranges that match the consistent hash ring’s partitions, so every node structures their trees the same way.

\textbf{anti\_entropy\_scheduler:} This part decides when and which two replicas should compare their data. It looks at how long it’s been since they last synced and picks pairs that are likely out of date.

\textbf{repair\_executor:} When differences are found, this component fetches only the keys or chunks that don’t match and applies the necessary updates to bring the replica up to date.


\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}

The repair process starts when the scheduler picks two replicas to check. They exchange the root hash of their Merkle trees. If the hashes don’t match, they go down into the tree step by step to find the exact subtree where the difference lies. Then, they request the specific list of keys that differ. Only those keys are sent from the newer copy to the older one. Once the update is applied, the replica’s state is updated to “healthy,” showing it’s back in sync.

\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

Merkle trees are built over fixed data ranges that match the hash ring’s partitioning, so all nodes agree on what each tree covers.

The same hash function and tree structure (like branching factor) are used everywhere, so trees can be compared correctly between different nodes.

After a repair session, the replica metadata is updated. This helps future reads know which copies are fresh and reliable.

\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}
We design the Merkle tree system as a reusable module with three main functions:

\textbf{generate:} builds a tree from a snapshot of a data segment

\textbf{serialize:} turns the tree into a format that can be sent over the network

\textbf{diff:} compares two trees to find differences
The trees are built on stable chunks of data that match how the hash ring divides up keys, so every node processes the same ranges.

Tree generation runs in the background on each datastore, so it doesn’t slow down regular reads and writes. We add throttling to limit how much bandwidth it uses important in low-resource environments.

Next, we implement the anti-entropy protocol:

Two replicas start a repair session and agree on which key range to check. They exchange the root hash of their Merkle trees. If they don’t match, they go deeper comparing child nodes step by step until they find the exact key-level differences. This process handles empty or missing ranges gracefully, avoiding errors when one side has no data.

When mismatches are found, the repair executor kicks in. It only requests the keys that are missing or outdated. Before applying updates, it checks version numbers and uses rules like last-write-wins or vector clock comparison to resolve conflicts safely.

After repair, the system updates the replica’s metadata:

- It marks the replica as up-to-date

- It clears any pending hinted handoff entries for that node, since the data is now current. 

This helps keep the system clean and avoids unnecessary retries.

Finally, we write simulation and integration tests that create realistic problems like dropped updates or stale replicas and then check that:

- The repair process fixes them correctly

- Only small amounts of data are transferred (not full copies)

- Normal operations like reading and writing continue without being blocked during repair



\subsection{Summary}


We build the system step by step. First, we set up decentralized routing so any node can find where data should go no central server needed. Then, we add quorum-based replication, which means each piece of data is stored on multiple nodes, and we only need a certain number of them to respond for reads and writes to succeed. This gives us durability without requiring every node to be online. Finally, we add Merkle tree-based repair (anti-entropy) to automatically fix differences between replicas, but only sync what’s missing saving bandwidth.

This step-by-step approach turns the architecture into working code in a clear way. Each part creates reusable components and shared metadata that the next part can use. Because everything is built and tested in stages, the risk stays low, and we can check each piece works before moving on.

The result is a system that stays reliable even when nodes fail, doesn’t waste network resources, and has no single point of failure  perfect for edge environments where connections are weak or unstable.


\section{System-aligned design flow}

\subsection{Implementation order the “why first” story}

 We build this system in a specific order so we don’t waste time, avoid confusion, and make sure each step rests on solid ground.

First comes decentralized data mapping because before we can replicate data safely, everyone needs to agree on where it should go. If each broker uses a different method to pick storage nodes, then under network changes or node failures, they might send the same data to completely different places. That leads to chaos: lost writes, inconsistent reads, no real durability. So we start by giving every broker the same shared understanding of the system.

Using gossip and consistent hashing, each broker builds an identical view of the network a hash ring that maps keys to datastores. This way, any broker can independently compute the same list of responsible nodes for any key, just by running a function locally. No coordination needed. Even during partitions, each side keeps working with its current view. This stable placement layer becomes the foundation for everything else.

Once we have agreement on where data belongs, we move to multiple redundant copies. Now that we know which three (or more) datastores should hold each key, we can safely write to all of them at once. We use quorum rules wait for W out of N confirmations so the system stays available even if one replica is slow or down. We also add hinted handoff to catch up later when failed nodes return. This turns what was once a single point of failure into a durable, fault-tolerant pipeline.

Only after replication do we add efficient incremental recovery using Merkle trees. Why? Because there’s nothing to repair until you have multiple copies that can diverge. Before replication, there’s only one source no drift. After replication, yes, the system works correctly thanks to quorums and hints but small inconsistencies can still build up silently over time. A node might miss updates, or corruption could go unnoticed.

That’s where Merkle-based repair comes in. Instead of copying entire datasets, nodes compare compact hash trees. When they find differences, they sync only the missing keys. This keeps bandwidth low and makes healing automatic, quiet, and efficient perfect for emergency networks with limited connectivity.

So the whole journey follows a clear path:
Map → Replicate → Heal

Each phase sets up the next. Each avoids rework. And together, they create a datastore layer that’s not just resilient, but built to last even when the network around it falls apart.

\subsection{How this fits into UCP/REC as it exists today}


This new replication system isn’t a rewrite it’s an evolution of what’s already there. REC already has a clean structure with four clear roles: clients, brokers, executors, and datastores. That doesn’t change. We’re not starting over; we’re building on top of what works.

Right now, brokers act as traffic coordinators. They receive jobs from clients, assign them to executors, and help manage where data goes. Datastores are simple they just store files or results under a name, like a key-value store. Clients submit work and wait for results. Executors run the actual tasks using WebAssembly.

Our goal isn’t to disrupt this. It’s to make the storage layer underneath more resilient without changing how anyone uses the system.

So what actually changes?

Brokers get smarter and they already route things now they also understand where data should live. 

We add two new layers:

A placement layer that uses the consistent hash ring and gossip to figure out which datastores are responsible for each key.
A replication layer that sends writes to multiple stores, waits for a quorum (W), handles timeouts, and tracks missed updates with hinted handoff.
None of this affects their core job. They still talk to clients and executors the same way. But behind the scenes, they’re no longer sending data to just one place they’re managing durability across many.

Datastores also grow up a bit. Right now, they just save blobs. Now, they need to keep track of more than just content. They start storing version vectors (or at least timestamps) so we can tell which update is newer when conflicts happen. They also build and maintain Merkle tree segments over their data, so they can prove what they have and compare themselves to others during repair. And they expose small endpoints so other nodes can request hashes or send missing keys.

We also introduce background processes that quietly keep the system healthy:

Gossip stabilizes membership who’s online, who just joined, who’s gone.
The hinted handoff queue replays missed writes when failed nodes come back.
An anti-entropy scheduler runs regular checkups between replicas, triggering Merkle-based repairs when needed.
But here’s the best part: clients don’t notice any of this. Their interface stays exactly the same. They still upload a job bundle, get a result callback, and retrieve outputs by key. Under the hood, the data is safer, more available, and self-healing — but to them, nothing changed.

Even executors stay untouched. They still pull inputs from nearby datastores, run WASM code, and push results back through the broker. The only difference is that the data they read and write now lives in multiple places and survives failures.

In short: we’re not redesigning REC. We’re strengthening its bones while keeping its skin the same. The system becomes resilient not by adding complexity everywhere, but by making quiet upgrades where they matter most brokers and datastores so everything else can keep working as intended.



\subsection{Phase 1: Decentralized mapping
}
\label{Phase 1: Decentralized mapping}
\textbf{We introduce:
}

membership\_gossip: lightweight heartbeats, join/leave, capability notes (storage size, load).


hash\_ring: consistent hashing with virtual nodes; each broker stores the same ring snapshot (versioned).

\textbf{Effect:} every broker can derive the same preference list for a key offline. If a broker dies, the rest don’t stall. If the network partitions, each side keeps a locally consistent placement until reconnection.

\textbf{Write flow at this point:} Client → Broker → Single datastore (still no real replication yet). The difference is we could scale horizontally later without rewriting this part.

\subsection{Phase 2: Multiple redundant copies}
\label{Phase 2: Multiple redundant copies}


\textbf{We widen the write path:}

\textbf{New broker pieces:}

replication\_controller: takes a key + value + policy, fans out writes to N target datastores (from the ring).

quorum\_tracker: collects acks, enforces W threshold and timeout logic.

hinted\_handoff\_queue: records writes that missed a target due to temporary unreachability.

consistency\_policy: holds (N, W, R) per data class (e.g., job results vs. reference datasets).

\textbf{Datastore changes:}

Idempotent PUT with version metadata (e.g., \{key, value, version\_vector, checksum\}).
Optional digest update per key-range (will feed Merkle later).

\textbf{Reads:}

Broker queries R replicas (could be 1 initially, then raise to 2) and selects the freshest (vector dominance or timestamp if we start simple).
If it sees a lagging replica, it can opportunistically “read repair” by writing the latest back.

\subsection{Phase 3: Efficient incremental recovery
}

Now some nodes may have spent time offline; hinted handoff catches recent changes, but older drift or partial replay might remain.

\textbf{We add to datastores:}

merkle\_index: a tree per key-range (range = virtual node slice). Internal nodes are hashes of children; leaves summarize small key buckets.

range\_repair\_executor: given a diff spec (“these leaves differ”), fetch missing or outdated keys.

anti\_entropy\_scheduler: rotates through replica pairs (or triples) so each range gets checked periodically (more aggressively for recently recovered nodes).




The repair protocol starts by comparing summaries instead of data. Two replicas that want to check consistency first exchange the root hash of their Merkle tree for a given key range. If the hashes are the same, they know all the data under that range matches no further action is needed.

If the root hashes differ, it means there’s some mismatch somewhere below. The replicas then go one level deeper and compare the hashes of the child nodes (with a fan-out factor, say 16). They repeat this step-by-step descent, only following branches where the hashes don’t match.

They keep going until they reach the leaf level the smallest chunks of data. At this point, they can identify exactly which keys are missing or different on one side.

Then, only those specific keys are transferred from the newer replica to the older one. Once applied, the repaired node updates its local values and rebuilds the Merkle tree upward, recalculating each parent hash up to the root.

This way, the system fixes inconsistencies using minimal bandwidth just a few hashes and the actual differences making it efficient and safe for use in low-bandwidth, high-latency environments like emergency networks.




\subsection{A real end-to-end example}

Let’s follow what happens when a job completes say, a drone finishes analyzing live video from a disaster zone. The executor running on the device finishes the WebAssembly task and sends a message to its local broker: “The result is ready. Key: flood\_map.”

The broker takes that key and runs it through a hash function. Using its local copy of the consistent hash ring built from gossip messages about which datastores are alive it computes the preference list: [DS-A, DS-C, DS-F]. These are the three nodes that should store this data. No coordination needed; every broker would get the same result.

Now, the replication controller steps in. It sends the result to all three datastores at once. DS-A and DS-C respond quickly with confirmations. That’s two out of three  enough to meet the write quorum (W=2). So the broker marks the write as successful.

But DS-F doesn’t answer. Maybe it’s moving out of range or temporarily offline. Instead of failing the operation, the system records a hint: “DS-F missed update to flood\_map.” This goes into the hinted handoff queue, waiting for DS-F to come back.

Meanwhile, the client (or the next job in the pipeline) gets a success response along with a version vector like {v: [b1=42]} so everyone knows this is the latest version.

Later, DS-F reconnects. Gossip detects it’s back online, and the broker triggers the hinted handoff queue. The missing write is replayed quietly in the background. DS-F receives the data, saves it, and updates its Merkle tree.

Then comes the nightly anti-entropy pass. The scheduler pairs up DS-A and DS-F to check consistency in their shared key range. They exchange root hashes. Most match except one leaf, corresponding to flood\_map. Only that small part differs.

So they request just that one key. The newer value is transferred a tiny amount of data and applied. The Merkle tree is updated upward, and now both replicas are fully in sync.

From now on, any read will see three consistent copies. No confusion. No need for read repair. The system healed itself without anyone noticing.

And if there had been a network partition between step 3 and 6 say, a bridge collapse splitting the city  both sides could still have accepted writes using the nodes they could reach. When the link returned, repair slowly closed the gap. Data wasn’t lost. It converged.

This is how UCP keeps working not by avoiding failures, but by expecting them, planning for them, and healing quietly afterward.

One of the best things about this three-phase approach is that it’s safe to roll out  even in a live system where failures can’t be tolerated. We’re not making big, irreversible changes all at once. Instead, each phase builds on the last, and we can test and control everything step by step.

\subsection{Module “personalities”}


\begin{itemize}
    \item Broker “brain”: decides who (ring) + how many succeed (quorum) + who missed (hints).
    \item Datastore “memory + mirror”: stores values, indexes them into Merkle ranges, cooperates in diff sessions.
    \item Scheduler “custodian”: balances repair cadence vs. foreground latency (can deprioritize busy ranges).
    \item Client “unaware app”: same contract; higher reliability emerges underneath.
\end{itemize}


\subsection{Summery of phases}


We start by teaching all brokers to speak the same language when it comes to data placement. Right now, if two brokers are asked where a key should go, they might give different answers especially during network changes. That’s dangerous for replication later. So first, we make sure everyone agrees.

They learn this through gossip: lightweight messages that spread across the network, telling each broker who’s alive, who just joined, and who’s gone. Based on that list, every broker builds the same consistent hash ring a shared map of which datastores are responsible for which keys. No central directory. No single point of failure. Just local computation using global rules.

This doesn’t change how data flows yet. But it sets the foundation: now, any broker can independently decide where data belongs even during partitions. This simple agreement layer is cheap to run, invisible to clients, and absolutely essential.

Once placement is stable, we move to replication not randomly, but with discipline. Instead of hoping data survives, we make it survive by design.

Every write now goes to N replicas (say, three), chosen from the preference list. The broker waits for W acknowledgments (say, two) before saying “done.” Fast, reliable, and fault-tolerant.

If one replica is down? We don’t block. We record a hint a note that says, “this node missed the update” and try again later when it comes back. Clients still get a simple “success” response. But behind the scenes, the broker isn’t just forwarding; it’s running a small state machine that ensures durability, no matter what.

Still, over time, things drift. Even with hints, some differences sneak in maybe due to long outages or silent failures. We could resync everything, but that would flood weak emergency links. So instead, we accept a truth about distributed systems: drift happens and plan for it calmly.

We use Merkle trees as fingerprints of data. Periodically, replicas compare tree roots. If they match, great. If not, they zoom into the mismatched branches going deeper until they find exactly which small group of keys differ. Then, only those keys are transferred.

No full syncs. No wasted bandwidth. Just surgical fixes.

When a node returns after being offline, it catches recent writes through hinted handoff, then fills in older gaps through targeted anti-entropy repair. It heals quietly, without overwhelming the network.

By the end, we have a layered system:

\begin{itemize}
    \item Mapping gives us a shared understanding of intent where data should be.
    \item Replication makes that intent durable writing it to multiple places.
    \item Repair keeps reality aligned with intent over time healing silently when things go off track.
\end{itemize}
And the best part? The executor and job pipeline don’t need to change. They keep doing what they’ve always done run tasks, read inputs, write outputs. But now, under the surface, the storage layer handles failures, repairs itself, and keeps data safe so they can focus on results.

We didn’t add complexity everywhere. We built strength where it matters. And because each phase rests on the last, the whole system grows more resilient not all at once, but step by step, like a well-built machine.

