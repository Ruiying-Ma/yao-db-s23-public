# Project3: Optimizer
## Preparations
- A predicate is divided by AND into several expressions, i.e., 

    ```
    predicate = (expr_1) AND (expr_2) AND ... AND (expr_n)
    ```
- An expression is of tree sturcture, i.e., 

    ```
            op
           /  \
          ch1 ch2
    ```
    where ch1 and ch2 are also expressions.
## Logical Optimizer
Always-fired rules: when traversing the naive parse tree, check whether the current node can do (1) Pushdown Join Predicate Rule (2) Pushdown Filter Rule (3) Convert SeqScan to RangeScan Rule (in order!). If can, then do these rules. Then move to the next plan node (either `ch_` or `ch2_`). 
### Pushdown Join Predicate Rule
- Goal: For all expressions in `JoinPlanNode`'s predicate, find those that only touch the left set of tables of the node and push them down (the same for the expressions that only touch the right set of tables of the node).
- Implementation: we only need to create `FilterPlanNode` as the root of left/right subtree of `JoinPlanNode` (i.e., the left/right child of `JoinPlanNode`) if we need to do predicate pushdown. 

    Hence we simply check whether an expression in `JoinPlanNode`'s predicate 
    - is of the form `ColumnExpr_l = ColumnExpr_r`, and
    - satisfies `JoinPlanNode->ch_.table_bitset_`$\cap$`ColumnExpr_1`$ = \phi$
    - satisfies `JoinPlanNode->ch2_.table_bitset_`$\cap$`ColumnExpr_r`$ = \phi$

    The remaining task of predicate pushdown is transformed to pushing down the filters (as follow).
### Pushdown Filter Rule
- Goal: For all expressions in `FilterPlanNode`'s predicate, try the best to combine them with the predicates of its descendents (e.g., `JoinPlanNode`, `SeqScanPlanNode`, `OrderByPlanNode`...), such that at last only `PrintPlanNode` or `LimitPlanNode` can have `FilterPlanNode` above it. 
- Remark: `LIMIT` and `WHERE` cannot commute. That's why we allow `LimitPlanNode` to have `FilterPlanNode` above, since at that time we cannot further pushdown the filter.
- Implementation: Check the type of the child node: 
    - `DistinctPlanNode` or `OrderByPlanNode`:  
        - Exchange itself with its child node
    - `FilterPlanNode` or `JoinPlanNode` or `SeqScanPlanNode` or `RangeScanPlanNode` or `HashJoinPlanNode`: 
        - Append predicate to its child's predicate
        - Delete itself
    - `ProjectPlanNode`: 
        - Apply `ProjectPlanNode`'s output_exprs on its own predicate (to create new predicate)
        - Exchange itself with its child node
    - `AggregatePlanNode`:
        - Apply `AggregatePlanNode`'s output_exprs on its own predicate (to create new predicate)
        - Append this new predicate to `AggregatePlanNode`'s group_predicate (`HAVING` clause)
        - Delete itself
### Convert SeqScan to RangeScan Rule
- Goal: For all expression in `SeqScanPlanNode`'s predicate, if there exist one that restrict the range of the primary key of the to-be-scanned table, then convert this node to `RangeScanPlanNode`
- Meaning: `RangeScanPlanNode` search the table using the index. In our project, the index is built on the primary key of the table. Hence if there is some bound on the primary key, we can utilize the index to narrow the range we need to scan, thus speeding up our DB.  
- Implementation: Check whether an expression is of the form 
    ```
    ColumnExpr op LiteralExpr
    ```

    where `op` is determined to be `EQ`, `LT`, `LEQ`, `GT`, `GEQ`.
## Cost-based Optimizer

### HyperLL
- Estimate the **Number of Distinct Values (NDV)** of a column.
- Algorithm: 

    ```
    Prepare hash function f
    Prepare buckets b_1, ..., b_n
    For a value v newly inserted
        Calculate x = f(v)
        Use the first log_2(n) bits of x to decide which bucket b_i v should be in
        Use the remaining bits of x to do hyperlog in this bucket
            Count the number k of leftmost 0s in these remaining bits
            Update b_i's data to k if k is greater than the current data of b_i
    After all values are inserted, a * n * n * harmonic_mean_denominator is the estimated NDV.
    ```
    Here $harmonic\_mean\_denominator = \frac{1}{\frac{1}{2^{l_1+1}} + \frac{1}{2^{l_2+1}} +\cdots + \frac{1}{2^{l_n+1}}}$, where $l_i$ is the data bucket $b_i$ holds.
### CountMinSketch
- Estimate the **Counts** of a value of a column (i.e., the number of duplications of this value in this column).
- Algorithm

    ```
    Prepare hash functions f_1, ..., f_n
    Prepare buckets b_1, ..., b_m
    Prepare an m x n table
    For a value v newly inserted
        Calculate x_1 = f_1(v), x_2 = f_2(v), ..., x_n = f_n(v)
        For i = 1 : n
            table[x_i % m][i] ++
    After all values are inserted, an arbitrary value k is estimated to occur for min_i{table[f_i(k) % m][i]} times
    ```
### Dynamic Programming
Suppose the set of all tables a SQL will touch is $T = \{T_0, \cdots, T_{n-1}\}$. We can use an $n$-bit integer $S$ to denote a subset of $T$: for $S = (S_{n-1}S_{n-2}\cdots S_{0})_2$, if $S_i = 1$, then $T_i$ is included in the subset, otherwise $S_i = 0$ and $T_i$ is not in this subset. 

For each subset $S$ of $T$, we need to record: 
- The **size** (number of tuples) of the output joining all tables in $S$ (take predicates into consideration)
- For each column $i$ of the output of joining $S$, we record its **NDV$_i$**.
- The optimal **cost $F$** of joining all tables in $S$
- When reaching the optimal cost, the last step of joing is to join the subset $H$ and $S \oplus H$ (XOR). We record this **$H$**.

1. Base Case: Estimate the cost of seqscanning one table $A$.
    - **cost**: #tuple of the to-be-scanned table
    - **size**: Find expressions of the form $A.col = value$. Use CountMinSketch to shrink the original #tuple. Find expressions of the forom $A.col > / < / \geq / \leq value$. Use $max^A, min^A$ to shrink the original #tuple.
    - **NDV$_i$**: same as the original statistics of $A$. If meet $A.col_i = value$ predicate, update it to 1.
    - **$H$**: $(0\cdots 010\cdots0)_2$ (itself)
2. DP: Estimate the cost of joining two subsets.

    - For subsets with $k$ tables: use DFS to find all such subsets
        - For a subset $S$ with $k$ tables: use DFS to find all $H$ such that $H\subsetneq S$ and $H$ is at least half as large as $S$. 
    - **cost**: cost($H$) + cost($S\oplus H$) + the cost of joining two subsets $H$ and $S\oplus H$
        - the cost of joining two subsets $H$ and $S\oplus H$ can be the cost of HashJoin (about size($H$) + size($S\oplus H$)) or NestedLoopJoin. We need to check the predicates for expression 

            ```
            A.col = B.col
            ```
        
            where $A\in H$ and $B\in S\oplus H$. Only if such expression exists can we use hash join.
        - Let the cost of NestedLoopJoin to be extremely large. In the project, I set it to `SIZE_MAX`. (A must! Otherwise the cost-based plan will apply NestedLoopJoin, which is extremely slow). Make sure that we apply HashJoin at our best.
        - Always let the small subset to be the build side.
    - **size**: Check the predicates for expression
            ```
            A.col_i = B.col_j
            ```
            where $A\in H$ and $B\in S\oplus H$. Then use the size and NDV's of $H$ and $S\oplus H$ to estimate: $\frac{N^H\cdot N^{S\oplus H}}{\max\{NDV^{H}_i, NDV^{S\oplus H}_j\}}$
    - **NDV$_i$**: Check the predicates for expression
            ```
            A.col_i = B.col_j
            ```
            where $A\in H$ and $B\in S\oplus H$. Then set $NDV^{H}_i, NDV^{S\oplus H}_j$ to be the same (same as the smaller one).
    - **$H$**: the $H$ that makes the cost the smallest. We always let $H$ to contain fewer tables than $S\oplus H$, thus making sure the build side is always smaller if we can apply HashJoin.
3. GenPlan: Now we can trace back from $(11\cdots 1)n$ to get the new plan. When recursively doing plan generating, we do not insert any predicate to plan nodes, and we set all basis nodes to be `SeqScanPlanNode`. Then we insert all predicates into the highest `JoinPlanNode`. Then we use `LogicalOptimizer::Optimize` and `ConverToHashJoin` to further push down predicates, convert `SeqScanPlanNode` to `RangeScanPlanNode` and convert `JoinPlanNode` to `HashPlanNode`. 

## Results
```
OptimizerTest.IntegerRangeScanTest
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:484]: Use: 6.470784759 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:559]: Use: 6.035204791 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:574]: Use: 0.00098137 s

OptimizerTest.StringRangeScanTest
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:484]: Use: 0.832205003 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:559]: Use: 0.758506188 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:574]: Use: 0.000829468 s

OptimizerTest.FloatRangeScanTest
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:484]: Use: 2.491549852 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:559]: Use: 1.81482037 s
[info][TestPKRangeScan@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:574]: Use: 0.000892043 s

OptimizerTest.JoinCommuteTest
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:188]: Insert into B uses 0.001241033 s.
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:193]: Insert into A uses 2.969398373 s.

OptimizerTest.JoinAssociate4Test
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:337]: Use: 0.031501995 s

OptimizerTest.JoinAssociate5Test
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:409]: Use: 1.81835074 s
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_opm.cpp:423]: Use: 0.189087582 s

Benchmark.JoinOrder10Q1     // std: 1.9
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_job.cpp:264]: Use 4.799842479s

Benchmark.JoinOrder10Q2     // std: 1.7
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_job.cpp:279]: Use 2.927640096s

Benchmark.JoinOrder10Q3     // std: 2.1
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_job.cpp:294]: Use 3.536935319s

Benchmark.JoinOrder10Q4     // std: 1.0
[info][TestBody@/home/autograde/autolab/p3optimization-handout/test/test_job.cpp:310]: Use 2.398707087s
```