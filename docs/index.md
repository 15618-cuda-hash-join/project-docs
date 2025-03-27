# Proposal

Zhiping Liao (zhiping2@andrew.cmu.edu), Siyuan Lin (slin3@andrew.cmu.edu)

# Summary

We are going to implement efficient hash join operators using NVIDIA CUDA. We will focus on OLAP workloads and evaluate select-project-join queries on TPC-H benchmarks by taking advantage of GPU architecture and parallelism.

# Background

Join operators are one of the most common physical operators in database management system. In the meantime, they are also one of the most expensive operators, attracting great interests to make them efficient. A join operator typically combines rows from two or more tables with given predicates fetching rows conditionally. It allows efficient data retrieval and problem-solving in complex business queries.

Most joins are equi-joins, which use only equality comparisons in the join-predicate [4]. Hash joins are the most important equi-join physical implemenations in the analytical world and are one of the most well-studied among all join algorithms. It motivates us to focus on implementing effcient hash join by taking advantage of GPU architecture. 

Hash join typically involves two steps: build phase and probe phase.

- Build phase will construct the hash table by inserting keys from the build table.
- Probe phase will iterate the probe table and try to find a match in the hash table, if there is a match, the emit the tuple.

While join is extensively studied under CPU context in mainstream databases, GPU with its massive parallelism and high memory bandwidth, offers great potential for speedup for join operator. GPU with SIMT execution model could fire hundreds of thousands of threads  simultaneously. Thus, both build and probe phase in hash join could be implemented by utilizing parallel threads to construct the hash table and probe the hash table with correct synchronizations.

# The Challenge

### Challenge 1

The first challenge would be to have correct and efficient implementations of GPU hash table.

During the build phase, CUDA threads are concurrently inserting entries into a hash table. The table should implement correct synchronizations. Here, atmoic operations are encouraged due to less overhead.

Moreover, since keys are randomly distributed in the hash table, how to efficiently exploit GPU memory hierarchy also poses a challenge, as global memory is far slower than cache or shared memory.

### Challenge 2

The second challenge would be to ensure efficient work assignments and fair load balance for each thread.

During the probing phase, each thread would need to probe a different key into the hash table, if there is a match, then the thread would emit the tuple into output. It will lead to high thread divergence as in the same warp some threads would hit while others would miss. Moreover, coalesced memory access would become difficult as keys are randomly distributed inside the hash table, and adjacent threads would very unlikely access non-adjacent memory.

### Challenge 3

The third challenge would be scaling to a large dataset.

GPU memory is limited. Even on a cutting-edge H100, its memory would be 80GB. It is big but would not be enough when handling real-world dataset where tables could be petabytes large. In this way, it requires GPU out-of-core execution to multiple iterations to move dataset into main memory, process, and store results back. Under this scenario, how to efficiently divide dataset into chunk and how to reduce data transfers would be great challenge as data movement between CPU and GPU would be very slow compared to GPU memory bandwidth (450GB/s vs. 3TB/s on H100 NVLink).

# Resources

We will start codebase with libraries for GPU data structure and algorithms provided by NVIDIA [1, 2], such as hash table implementations in [cuCollections](https://github.com/NVIDIA/cuCollections), and join algorithms in [cuDF](https://github.com/rapidsai/cudf). We also want to incorporate an SQL frontend (Apache Calcite, DuckDB, PostgreSQL) which transforms SQL queries into physical plans.

From our side, we will implement data loader into the GPU memory and GPU kernels for join evaluation from scratch. Customized kernels offer a more degree of freedom and enable further optimizations. Besides, we will consult to a previous overview [3] of GPU accelerated database, looking for hints and insights on GPU database execution.

The development compute resources would be GHC machines with RTX 2080, and the benchmark compute resources would be PSC machines with V100. More advanced machines have higher memory bandwidth and interconnect bandwidth, which allows for more space for optimizations.

# Goals and Deliverables

1. Build a simple but sufficient testing framework that 1) can parse and convert the queries into physical plans, 2) load the data to join into GPUs and 3) execute and evaluate the hash join implementation. We will incorporate open source libraries to streamline it.
2. Implement a baseline inner hash join implementation leveraging cuCollections and cuDF. Complete an end-to-end testing of our framework and make sure things work.
3. Refine inner joins with customized table building, probing strategies, materialized output or other improvements. This will be the major part of our project. We will profile the performance after applying each technique, reason about its fitness in analytical workloads, explore potential improvements and narrow down to a set of optimizations with the best performance.
4. **Bonus**: implement outter joins to support more usecases.
5. **Bonus**: implement out-of-core execution for hash join when tables are larger than GPU main memory.

# Platform Choice

We will choose Linux with NVIDIA GPUs with modern CUDA support as our platforms. The core implementation of the join algorithms will be written in CUDA C++. We would utilize GHC machines (equipped with RTX 2080) as our development environment, and consider PSC (V100) / AWS instances to be our benchmark environment.

# Schedule

| Date | Task |
| --- | --- |
| Mar 27 - April 2 | Build the framework supporting query parsing, data loading and benchmarking. |
| April 3 - April 9 | Implement and benchmark the baseline hash join implementation, looking for rooms to improve. |
| April 10 - April 16 | Prepare the milestone report. Customize the hash table implementation with at least 2 techniques. Benchmark them and looking for further improvements. |
| April 17 - April 23 | Devise and benchmark another 2 techniques. Narrow down the optimizations for the best performance. |
| April 24 - April 28 | Write the final report. In the mean time, assess the attainablity of a bonus point, implement and discuss its performance characteristics. |

# References

1. [https://developer.nvidia.com/blog/maximizing-performance-with-massively-parallel-hash-maps-on-gpus/](https://developer.nvidia.com/blog/maximizing-performance-with-massively-parallel-hash-maps-on-gpus/#:~:text=For%20collision%20resolution%2C%20linear%20probing,the%20hash%20table%20insert%20function.&text=Looking%20up%20the%20associated%20value,be%20resident%20within%20the%20table)
2. [https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2019-s9557/](https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2019-s9557/)
3. [https://arxiv.org/abs/2406.13831v1](https://arxiv.org/abs/2406.13831v1)
4. [https://en.wikipedia.org/wiki/Join_(SQL)](https://en.wikipedia.org/wiki/Join_(SQL))