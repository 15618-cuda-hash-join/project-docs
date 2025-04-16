# Milestone Report

Zhiping Liao (zhiping2), Siyuan Lin (slin3)

# Revised Schedule

| Date | Task |  |
| --- | --- | --- |
| Mar 27 - April 2 | Looked for necessary open source components to simplify the data loading and query plan generation. Integreted them to a loading framework. | Data loading: Siyuan, Query and plan framework: Zhiping |
| April 3 - April 9 | Continued on building the framework. Implement and benchmark the baseline CPU hash join implementation, looking for rooms to improve. | Basic operators: Siyuan, Framework: Zhiping Liao |
| April 10 - 16 | Wrote the milestone report. Implemented and optimized the GPU-based hash join. | Report: Zhiping and Siyuan, GPU: Siyuan |
| April 17 - 23 | Devise and benchmark another 2 techniques. Narrow down the optimizations for the best performance. | Optimizations: Zhiping and Siyuan |
| April 24 - 28 | Write the final report. In the mean time, assess the attainablity of implementing a hash aggragtor on GPU. | Report: Zhiping and Siyuan |

# Current Progress

The progress we have made till the milestone generally involves several important points below:

- Prepared TPC-H dataset:
    - Cleaned data.
    - Transformed into `parquet` files for faster loading.
    - Loaded into C++ program using Apache Arrow.
    - Selected and rewrote a number of queries to suit our project’s need.
- Implemented fundamental database executor from scratch, including `TableScan`, `Filter`, `Join`, `Project`. Verified the correctness of these operators on CPU.
- Built database execution pipeline by reading a query plan and translating into corresponding physical operators:
    - Used DuckDB to generate query plans and table schemas.
    - Used `sqlglot` to parse SQL expressions in predicates and join conditions.
    - Curated and rewrote queries to limit them into the operators we chose.
    - Bound query plans and expressions to operators with given table schema.
- Zoomed in on the join operator and implemented a baseline GPU join operator using cuco `static_multimap` host API as starting point.
- Optimized the GPU join operator by switching to `static_multiset` to reduce extra data materialization costs and better and handy API usage.

# Revised Goals and Delieverables

As we were exploring the datasets and assessing the feasibility, we noticed several issues:

1. Semi join and outter join plans result in `DELIM` joins and scans: 3-way join operators and result reuse, which are non-trivial to support in our framework.
2. Aggregations are challenging to implement and optimize because 1) it requires a hash table just as normal joins, 2) only simple aggregation functions (`min`, `sum`, `avg`) can be implemented and need static dispatch to achieve ideal performance (one CUDA kernel per each aggregation function). We thus decide to choose aggregations as our bonus target.
3. Other expressions and operators would add significant amount of work but do not contribute to the goals of our projects. We agreed to drop their supports and continued to emphasis hash joins on GPU.
4. `libcudf` turns out to be closely couple with the cuDF dataframe representation and its complexity is way beyond our imagination. We settled on NVIDIA’s cuCo (cuCollections) as our references to implement hash join algorithms. However, `libcudf`'s hash join implementation does provide us with some insights for our design.

Based on our observations, we made a few changes to our goals and deliverables:

1. Hash join algorithms will only rely on cuco, not cuDF, as it provides sufficient functionality, customizability and better performance. 
2. Our further operator optimizations will mainly focus on join performance optimization, including but not limited to custom kernel optimizations, reduced data materialization, data movement and kernel execution overlapping.
3. We will also focus on data-loading or pre-processing before join, as data patterns will affect hash table insertion and lookup, which can indirectly affect cache friendliness and hit rate. This is important for performance speedup.
4. We change our **bonus points** to implement and refine hash aggregation with a fixed set of functions: `sum`, `min`, `max` and `avg`.
5. We want to reduce the supports for other operators and expressions. 

# Poster Session Plan

Our poster session will likely to demonstrate:

1. Background information about our project and why it is challenging;
2. The architecture of the project: what tools are used and how they are chained together;
3. The speedup and performance comparison between different versions;
4. A deep analysis of how we progressively diagnose the performance and arrive at the final solution.

# Preliminary Results

Currently as we only focus on join, so the SQL query for development and benchmark only involves one join, and it is sufficient for us to benchmark and improve on its performance. The query we focus on currently is a simplified TPCH Q21, which eliminates aggregation and sub-query, and only focus on join.

## Q21 Simplified Query

Setup:

- TPC-H dataset scale factor: 10.
- Machine: `g4dn.xlarge`, NVIDIA T4 Tensor Core, 4 vCPUs, 16 GB Memory.
- Environment: `amazon/Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 22.04)`
- Here we use cuco as the hash table library, and it provides host API that can be called on CPU, and devices API that can only be called inside a kernel.

```sql
select
  s_suppkey, l_orderkey
from
  supplier,
  lineitem l1,
where
  s_suppkey = l1.l_suppkey
  and s_nationkey = 1
```

|  | CPU Join | GPU Join with multimap | GPU Join with multiset |
| --- | --- | --- | --- |
| Time | 4.8s | 5.1s | 3.5s |

Analysis:

- For scale factor 10, the large table `lineitem` would contain 6M rows, and would be as large as 5GB.
- `static_multimap` is worse because it introduces extra data materialization costs due to the API, whenever there is match, it can only output the build table match, but not specifying probing matching. `static_multiset`  works as we can encode `(Key, row idx)` as the key for set, and it could give us correct results without extra looking up the probing table. (Some details are omitted here).
- There are still some extra data materialization costs and data movement that could be saved by careful optimizations. They take almost half of the time due to frequent CPU-GPU data transfer.

# Primary Concerns

We are concerned about the excessive complexity of our framework (data loading, plan building and expression evaluation). It now poses non-neglegible debugging overheads to us.