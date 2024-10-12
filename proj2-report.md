# Project 2: Executor
## Understanding `std::unqiue_ptr<T>`
`std::unqiue_ptr<T>` is an encapsulation of an original pointer pointing to type `T`. 
It can automatically `delete` the memory allocated (`new`) before when it dies (exits its lifecycle). 
- When constructing a unique pointer of an array, 
    ```c++
    std::unique_ptr<T[]> ptr (new int[size]);
    ```
    Need to add a `[]` in the template
- Can never copy a unique pointer. Can only move: 
     ```c++
    std::unique_ptr<T> new_ptr = std::move(old_ptr)
    ```
  
For more, see [reference (csdn)](https://blog.csdn.net/shaosunrise/article/details/85158249?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522168215216516800222814845%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=168215216516800222814845&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-85158249-null-null.142^v86^wechat,239^v2^insert_chatgpt&utm_term=unique_ptr&spm=1018.2226.3001.4187)

## Understanding `std::sort`
When defining the compare function `comp` of `std::sort` by ourselves, we can never let  `comp` return `true` if the compared values are equal. 
We can only make `comp(auto a, auto b)` return `true` when a<b.

## Implementation Framework
- Join Executor
  - `Init()`
    1. Init its children excutors
    2. Read in **all** its left child's tuples. **Copy** (must do this copy, because the contents of the pointer of the tuple it gets may change, as the child executor keeps refilling this pointer with different returned tuples) the tuples in a table `build_side_`.
  - `Next()`
    1. Read in a tuple from its right child.
    2. Scan the `build_side_` table from where it stops last time it was called to find a matching tuple according to the join predicate.
    3. If find one, immediately return the concatenation of these two matching tuples (need to allocate a new space to store this concatenated tuple).
    4. If cannot find one, read in the next tuple from its right child, back to step 1.
  
  Need to ensure that its output tuple must be in the form of StaticFieldRef. If its left input tuple is raw, then by using api `TUpleStore.Append(...)`, we can store them automatically in StaticFieldRef (i.e., deserialize). 
  If the right input tuple is raw, then we need to convert it into StaticFieldRef (`Tuple::Deserialize`) by hand when concatenating tuples.

  Zero-copy in Join Executor: only need to allocate space for the left child tuples and the output concatenated tuples. No need to save the right child tuples. 
- Hashjoin Executor
  - `Init()`
    1. Init its children excutors
    2. Read in **all** its left child's tuples. **Copy** the tuples in a table `hash_table_` using hash function f.  
  - `Next()`
    1. Read in a tuple from its right child.
    2. Find the hash key of this tuple using hash function f same as in `Init()`.
    3. Scan the bucket of that hash key from where it stops last time.
    4. If find one, immediately return the concatenation of these two matching tuples (need to allocate a new space to store this concatenated tuple).
    5. If cannot find one, read in the next tuple from its right child, back to step 1.

  Need to ensure that its output tuple must be in the form of StaticFieldRef. Same as Join executor.
- Aggregate Excutor
  - `Init()`
    1. Init its child excutor
    2. Read in **all** its child's tuples. Every time read in a tuple, use `group_predicate`(GROUP BY clause) to find its group (by hashing).
    3. If the group already existed, `Aggregate()` and update the group's AggregateIntermediateData.
    4. If the group doesn't exist, create a new group, `FirstEvaluate()` and **copy** this first tuple into `stored_param` for future use (used by SELECT clause). 
  - `Next()`
    1. Get a group
    2. Check whether the group satisfies the pedicate (HAVING clause)
    3. If yes, then compute the aggregate using `LastEvaluate` and the `stored_param` to return the result tuple. The format is determined by `output_exprs` (SELECT clause).
    4. If not, get the next group and back to step 1.

  Any exprs containing aggregations must generate its function using `AggregateExprFunction`. Both `ExprFunction` and `AggregateExprFunction` can handle raw and StaticFieldRef tuples. 
  So there is no need to check whether the input tuple is raw or not. And by using `TupleVector` on `stored_param`, these stored tuples will always be converted into StaticFieldRef form.

  Zero copy in Aggregate executor: only need to store the first tuple of a group. #group << #tuples.
- Order By Executor
  - `Init()`
    1. Init its child excutor
    2. Read in **all** its child's tuples and **copy** them.
    3. Sort using `std::sort`
  - `Next()`
    1. Return one tuple in order each time.
- Limit Executor
  - `Init()`
    1. Init its child excutor
  - `Next()`
    1. Discard the first `offset_` tuples
    2. Return the tuple if in range `[offset_, offset_ + limit_)`
- Distinct Executor
  - `Init()`
    1. Init its child excutor
    2. Maintain a hash table to record whether a tuple has appeared before
  - `Next()`
    1. Read in a tuple.
    2. If its hash key hasn't appeared, add this key into the hash table and return this tuple
    3. If its hash key already existed, read the next tuple and back to step 1.

  Here, *same* tuple is defined based on the SELECT clause.
## Results 
Performance
```
[==========] Running 14 tests from 6 test suites.
[----------] Global test environment set-up.
[----------] 4 tests from ExecutorJoinTest
[ RUN      ] ExecutorJoinTest.JoinTestNum10Table2
[       OK ] ExecutorJoinTest.JoinTestNum10Table2 (1 ms)
[ RUN      ] ExecutorJoinTest.JoinTestNum3e3Table2
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:320]: Use: 0.681342289 s           // TA: 0.576599203 s
[       OK ] ExecutorJoinTest.JoinTestNum3e3Table2 (1400 ms)
[ RUN      ] ExecutorJoinTest.JoinTestTable3
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:415]: Use: 0.036445832 s           // TA: 0.008260367 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:456]: Use: 1.385745746 s           // TA: 0.412558647 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:481]: 313689
[       OK ] ExecutorJoinTest.JoinTestTable3 (10018 ms)
[ RUN      ] ExecutorJoinTest.JoinTestTableN
[       OK ] ExecutorJoinTest.JoinTestTableN (559 ms)
[----------] 4 tests from ExecutorJoinTest (11979 ms total)

[----------] 3 tests from ExecutorAggregateTest
[ RUN      ] ExecutorAggregateTest.SmallAggregateTest
[       OK ] ExecutorAggregateTest.SmallAggregateTest (0 ms)
[ RUN      ] ExecutorAggregateTest.PolyAggregateTest
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:915]: Use: 3.366112358 s           // TA: 1.732347523 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:918]:
[       OK ] ExecutorAggregateTest.PolyAggregateTest (3386 ms)
[ RUN      ] ExecutorAggregateTest.StringAggregateTest
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:967]: Use: 2.504540808 s           // TA: 1.88561714 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:992]: 1868
[       OK ] ExecutorAggregateTest.StringAggregateTest (3341 ms)
[----------] 3 tests from ExecutorAggregateTest (6727 ms total)

[----------] 2 tests from ExecutorOrderByTest
[ RUN      ] ExecutorOrderByTest.SmallTest
[       OK ] ExecutorOrderByTest.SmallTest (0 ms)
[ RUN      ] ExecutorOrderByTest.BigTest
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1083]: Use: 2.445031191 s            // TA: 0.863839554 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1108]: Use: 1.43304308 s           // TA: 0.758876405 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1132]: Use: 0.794549474 s            // TA: 0.59437998 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1159]: Use: 2.033451283 s            // TA: 0.926390198 s
[       OK ] ExecutorOrderByTest.BigTest (14254 ms)
[----------] 2 tests from ExecutorOrderByTest (14254 ms total)

[----------] 2 tests from ExecutorLimitTest
[ RUN      ] ExecutorLimitTest.SmallTest
[       OK ] ExecutorLimitTest.SmallTest (1 ms)
[ RUN      ] ExecutorLimitTest.BigTest
[       OK ] ExecutorLimitTest.BigTest (868 ms)
[----------] 2 tests from ExecutorLimitTest (869 ms total)

[----------] 2 tests from ExecutorDistinctTest
[ RUN      ] ExecutorDistinctTest.SmallTest
[       OK ] ExecutorDistinctTest.SmallTest (1 ms)
[ RUN      ] ExecutorDistinctTest.BigTest
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1338]: Use: 0.417309158 s
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1354]: Use: 0.035111664 s
[       OK ] ExecutorDistinctTest.BigTest (878 ms)
[----------] 2 tests from ExecutorDistinctTest (879 ms total)

[----------] 1 test from ExecutorAllTest
[ RUN      ] ExecutorAllTest.OJContestTest
[info][TestBody@/home/2021010726/wing/test/test_exec.cpp:1461]: Use: 9.006986689 s            // TA: 3.484005504 s
[       OK ] ExecutorAllTest.OJContestTest (14067 ms)
[----------] 1 test from ExecutorAllTest (14067 ms total)

[----------] Global test environment tear-down
[==========] 14 tests from 6 test suites ran. (48775 ms total)
[  PASSED  ] 14 tests.
```

Tip: It is said that if storing the whole page (instead of page_id) in `Iter` of the bplus tree, the performance of the last test can be improved from 10s to 5s