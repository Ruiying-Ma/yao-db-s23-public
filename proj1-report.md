# Porject 1: SortedPage & BPlusTree
## SortedPage
### Structure:

Base: `Page`

Derived: 
- `PlainPage` : `MetaPage` of a BPlusTree
- `SortedPage`: `LeafPage` and `InnerPage` of a BPlusTree

### Implementation of `SortedPage`

- `InsertBeforeSlot`
  - input: 
    - `size`: the size of the to-be-inserted string_view
    - `slotid`: the to-be-inserted string_view is at `slotid` after the insertion
  - output:
    - `char *`: the raw pointer pointing to the address of the start of the to-be-inserted string_view
- `SplitInsert`
  - Splitting the page evenly, such that the original page size ~ split page size after splitting and inserting
  - Implemented by record the split page size dynamically, and begin to split as soon as the page is evenly split
  - Optimization: move the slots from old page to new page in increasing order of their slotid's
- `ReplaceSlot`
  - Naively, just delete and insert
  - For performance's sake, we record the `delta` of movements to directly implement replacement
- `SplitReplace`
  - Simply delete and insert

## BPlusTree
### Structure:

A BPlusTree <-> A table in database

```cpp
/* Level 0: Leaves
 * Level 1: Inners
 * Level 2: Inners
 * ...
 * Level N: Root
 *
 * Initially the root is a leaf.
 *-----------------------------------------------------------------------------
 * Meta page:
 * Offset(B)  Length(B) Description
 * 0          1         Level num of root
 * 4          4         Root page ID
 * 8          8         Number of tuples (i.e., KV pairs)
 *-----------------------------------------------------------------------------
 * Inner page:
 * next_0 key_0 next_1 key_1 next_2 ... next_{n-1} key_{n-1} next_n
 * ^^^^^^^^^^^^ ^^^^^^^^^^^^            ^^^^^^^^^^^^^^^^^^^^ ^^^^^^
 *    Slot_0       Slot_1                    Slot_{n-1}      Special
 * Note that the lengths of keys are omitted in slots because they can be
 * deduced with the lengths of slots.
 *-----------------------------------------------------------------------------
 * Leaf page:
 * len(key_0) key_0 value_0 len(key_1) key_1 value_1 ...
 * ^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^
 *        Slot_0                   Slot_1
 *
 * len(key_{n-1}) key_{n-1} value_{n-1} prev_leaf next_leaf
 * ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^
 *            Slot_{n-1}                      Special
 * The type of len(key) is pgoff_t. Note that the lengths of values are omitted
 * in slots because they can be deduced with the lengths of slots:
 *     len(value_i) = len(Slot_i) - sizeof(pgoff_t) - len(key_i)
 */
```

### Implementation
- `Insert`
  - input: key, value
  - Go down from the root to the leaf:
    - In `InnerPage`:
      - Pay attention to `special` of a innerpage: `special` may be 0. At this point, we need to create a child for it. The child may either be `InnerPage` or `LeafPage`. Don't forget to set the `special` of this innerpage.
      - Using `UpperBound(key)` to find the right child page
    - In `LeafPage`:
      - Using `LowerBound(key)` to find the target slot
  - If no need to split the leaf, just `InsertBeforeSlot`
  - If need to split the leaf
    - Solve leafpage's prev/next and its sibling's prev/next
    - Split the leaf
    - Solve leafpage's parent's `next` pointer
    - Insert the new slot into leafpage's parent, recursively go up
  - If eventually need to split the root
    - Update the metapage's root and level_num data
  - Update the tuple_num data in metapage
  - Using an array `path[level_num +1]` to record the path from the root to the leaf to implement the split process
- `Update`
  - Input: key, value
  - Go down from the root to the leaf:
    - Split & insert: same as that in `Insert`
    - Except that: 
      - no need to update tuple_num
      - when no way to go down, no need to create a new page, just return `false`
- `Get`
  - Input: key
  - Go down the tree to find the key's corresponding value
- `Delete`
  - Input: key
  - Go down the tree to find the key
  - If the leaf has only this slot, then free this leaf
    - Set this leaf's siblings' prev/next
    - If the leaf's parent points to this leaf with a slot
      - Delete that slot in parent too. 
    - If the leaf's parent points to this leaf with `special`
      - Set this `special` to 0.
    - If the parent has no child (parent may have no slot but still has a child), free this parent. Recursively go up.
  - If the root is deleted, create a new leaf node and update the root data in the metapage.
  - Udpate the tuple_num in the metapage.
- `Take`
  - Naively, Get and Delete
  - For performance's sake, use `Delete`'s implementation, except that we need to save the value we get this time.
- `Destroy`
  - Idea: level_traverse the tree from root the leaf. Every time when we get all children of a page into the queue, we free that page.
- `Scan`
  - For performance's sake, we should avoid getting leaf page every time we call `Next()`, since at most time, the iter will stay in the same page, and we only need to add the slot_id to move to the next slot. We can achieve this by maintaining the slot_num of the page the iter is pointing to inside the iter.


## Results
```
[----------] 104 tests from BPlusTreeTest
[ RUN      ] BPlusTreeTest.Basic1
[       OK ] BPlusTreeTest.Basic1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e1
[       OK ] BPlusTreeTest.RandInsertGet1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e2
[       OK ] BPlusTreeTest.RandInsertGet1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e3
[       OK ] BPlusTreeTest.RandInsertGet1e3 (4 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e4
[       OK ] BPlusTreeTest.RandInsertGet1e4 (32 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e5
[       OK ] BPlusTreeTest.RandInsertGet1e5 (315 ms)
[ RUN      ] BPlusTreeTest.RandInsertGet1e6
[       OK ] BPlusTreeTest.RandInsertGet1e6 (4286 ms) // benchamark: 3782ms   upperbound: 5673ms
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e1
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e1 (38 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e2
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e2 (0 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e3
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e3 (2 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e4
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e4 (21 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e5
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e5 (309 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue16B1e6
[       OK ] BPlusTreeTest.SeqInsertGetValue16B1e6 (3919 ms) // benchmark: 4290ms   uppderbound: 6435ms
[ RUN      ] BPlusTreeTest.SeqInsertGetValue1024B1e1
[       OK ] BPlusTreeTest.SeqInsertGetValue1024B1e1 (96 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue1024B1e2
[       OK ] BPlusTreeTest.SeqInsertGetValue1024B1e2 (2 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue1024B1e3
[       OK ] BPlusTreeTest.SeqInsertGetValue1024B1e3 (13 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue1024B1e4
[       OK ] BPlusTreeTest.SeqInsertGetValue1024B1e4 (143 ms)
[ RUN      ] BPlusTreeTest.SeqInsertGetValue1024B1e5
[       OK ] BPlusTreeTest.SeqInsertGetValue1024B1e5 (2019 ms) // benchmark: 1730ms   upperbound: 2595ms
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e1
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e1 (8 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e2
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e3
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e3 (3 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e4
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e4 (28 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e5
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e5 (347 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpdateGet1e6
[       OK ] BPlusTreeTest.RandInsertUpdateGet1e6 (5124 ms) // benchmark: 4748ms   upperbound: 7122ms
[ RUN      ] BPlusTreeTest.Delete1
[       OK ] BPlusTreeTest.Delete1 (38 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e1
[       OK ] BPlusTreeTest.RandInsertGetDelete1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e2
[       OK ] BPlusTreeTest.RandInsertGetDelete1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e3
[       OK ] BPlusTreeTest.RandInsertGetDelete1e3 (2 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e4
[       OK ] BPlusTreeTest.RandInsertGetDelete1e4 (18 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e5
[       OK ] BPlusTreeTest.RandInsertGetDelete1e5 (214 ms)
[ RUN      ] BPlusTreeTest.RandInsertGetDelete1e6
[       OK ] BPlusTreeTest.RandInsertGetDelete1e6 (2453 ms) // benchmark: 2148ms   upperbound: 3222ms
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e1
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e2
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e3
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e3 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e4
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e4 (15 ms)
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e5
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e5 (177 ms)
[ RUN      ] BPlusTreeTest.RandInsertDeleteAll1e6
[       OK ] BPlusTreeTest.RandInsertDeleteAll1e6 (2533 ms) // benchmark: 2368ms   upperbound: 3552ms
[ RUN      ] BPlusTreeTest.Scan1
[       OK ] BPlusTreeTest.Scan1 (39 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e1
[       OK ] BPlusTreeTest.RandInsertScanAll1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e2
[       OK ] BPlusTreeTest.RandInsertScanAll1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e3
[       OK ] BPlusTreeTest.RandInsertScanAll1e3 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e4
[       OK ] BPlusTreeTest.RandInsertScanAll1e4 (11 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e5
[       OK ] BPlusTreeTest.RandInsertScanAll1e5 (140 ms)
[ RUN      ] BPlusTreeTest.RandInsertScanAll1e6
[       OK ] BPlusTreeTest.RandInsertScanAll1e6 (2093 ms) // benchmark: 1929ms   upperbound: 2893.5ms
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e1
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e1 (41 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e2
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e2 (0 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e3
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e3 (1 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e4
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e4 (13 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e5
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e5 (152 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue16B1e6
[       OK ] BPlusTreeTest.SeqInsertScanAllValue16B1e6 (2014 ms) // benchmark: 1709ms   upperbound: 2563.5ms
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue1024B1e1
[       OK ] BPlusTreeTest.SeqInsertScanAllValue1024B1e1 (102 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue1024B1e2
[       OK ] BPlusTreeTest.SeqInsertScanAllValue1024B1e2 (1 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue1024B1e3
[       OK ] BPlusTreeTest.SeqInsertScanAllValue1024B1e3 (12 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue1024B1e4
[       OK ] BPlusTreeTest.SeqInsertScanAllValue1024B1e4 (127 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValue1024B1e5
[       OK ] BPlusTreeTest.SeqInsertScanAllValue1024B1e5 (1567 ms) // benchmark: 1241ms   upperbound: 1861.5ms
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e1
[       OK ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e1 (1 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e2
[       OK ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e2 (0 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e3
[       OK ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e3 (6 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e4
[       OK ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e4 (71 ms)
[ RUN      ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e5
[       OK ] BPlusTreeTest.SeqInsertScanAllValueMax1024B1e5 (760 ms) // benchmark: 613ms   upperbound: 919.5ms
[ RUN      ] BPlusTreeTest.LowerBound1
[       OK ] BPlusTreeTest.LowerBound1 (5 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e1
[       OK ] BPlusTreeTest.RandInsertLowerBound1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e2
[       OK ] BPlusTreeTest.RandInsertLowerBound1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e3
[       OK ] BPlusTreeTest.RandInsertLowerBound1e3 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e4
[       OK ] BPlusTreeTest.RandInsertLowerBound1e4 (21 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e5
[       OK ] BPlusTreeTest.RandInsertLowerBound1e5 (247 ms)
[ RUN      ] BPlusTreeTest.RandInsertLowerBound1e6
[       OK ] BPlusTreeTest.RandInsertLowerBound1e6 (3631 ms) // benchmark: 3338ms   upperbound: 5007ms
[ RUN      ] BPlusTreeTest.UpperBound1
[       OK ] BPlusTreeTest.UpperBound1 (41 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e1
[       OK ] BPlusTreeTest.RandInsertUpperBound1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e2
[       OK ] BPlusTreeTest.RandInsertUpperBound1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e3
[       OK ] BPlusTreeTest.RandInsertUpperBound1e3 (2 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e4
[       OK ] BPlusTreeTest.RandInsertUpperBound1e4 (20 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e5
[       OK ] BPlusTreeTest.RandInsertUpperBound1e5 (249 ms)
[ RUN      ] BPlusTreeTest.RandInsertUpperBound1e6
[       OK ] BPlusTreeTest.RandInsertUpperBound1e6 (3606 ms) // benchmark: 3286ms   upperbound: 4929ms
[ RUN      ] BPlusTreeTest.RandInsertScan1e1
[       OK ] BPlusTreeTest.RandInsertScan1e1 (41 ms)
[ RUN      ] BPlusTreeTest.RandInsertScan1e2
[       OK ] BPlusTreeTest.RandInsertScan1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertScan1e3
[       OK ] BPlusTreeTest.RandInsertScan1e3 (3 ms)
[ RUN      ] BPlusTreeTest.RandInsertScan1e4
[       OK ] BPlusTreeTest.RandInsertScan1e4 (39 ms)
[ RUN      ] BPlusTreeTest.RandInsertScan1e5
[       OK ] BPlusTreeTest.RandInsertScan1e5 (480 ms)
[ RUN      ] BPlusTreeTest.RandInsertScan1e6
[       OK ] BPlusTreeTest.RandInsertScan1e6 (6632 ms) // benchmark: 5262ms   upperbound: 7893ms
[ RUN      ] BPlusTreeTest.RandAllOperations1e1
[       OK ] BPlusTreeTest.RandAllOperations1e1 (41 ms)
[ RUN      ] BPlusTreeTest.RandAllOperations1e2
[       OK ] BPlusTreeTest.RandAllOperations1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandAllOperations1e3
[       OK ] BPlusTreeTest.RandAllOperations1e3 (4 ms)
[ RUN      ] BPlusTreeTest.RandAllOperations1e4
[       OK ] BPlusTreeTest.RandAllOperations1e4 (48 ms)
[ RUN      ] BPlusTreeTest.RandAllOperations1e5
[       OK ] BPlusTreeTest.RandAllOperations1e5 (516 ms)
[ RUN      ] BPlusTreeTest.RandAllOperations1e6
[       OK ] BPlusTreeTest.RandAllOperations1e6 (6314 ms) // benchmark: 4801ms   upperbound: 7201.5ms
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e1
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e2
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e2 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e3
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e3 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e4
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e4 (12 ms)
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e5
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e5 (146 ms)
[ RUN      ] BPlusTreeTest.RandInsertCloseOpenScan1e6
[       OK ] BPlusTreeTest.RandInsertCloseOpenScan1e6 (2141 ms) // benchmark: 1984ms   upperbound: 2976ms
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e1
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e1 (42 ms)
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e2
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e2 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e3
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e3 (9 ms)
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e1Value23333
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e1Value23333 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e2Value23333
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e2Value23333 (17 ms)
[ RUN      ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e3Value23333
[       OK ] BPlusTreeTest.RandInsertBlobCloseOpenScanDestroy1e3Value23333 (171 ms) // benchmark: 146ms   upperbound: 219ms
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e1
[       OK ] BPlusTreeTest.RandInsertDestroy1e1 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e2
[       OK ] BPlusTreeTest.RandInsertDestroy1e2 (0 ms)
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e3
[       OK ] BPlusTreeTest.RandInsertDestroy1e3 (1 ms)
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e4
[       OK ] BPlusTreeTest.RandInsertDestroy1e4 (11 ms)
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e5
[       OK ] BPlusTreeTest.RandInsertDestroy1e5 (141 ms)
[ RUN      ] BPlusTreeTest.RandInsertDestroy1e6
[       OK ] BPlusTreeTest.RandInsertDestroy1e6 (2032 ms) // benchmark: 1928ms   upperbound: 2892ms
[----------] 104 tests from BPlusTreeTest (55982 ms total) // benchmark: 49401ms   upperbound: 74101.5ms

[----------] Global test environment tear-down
[==========] 104 tests from 1 test suite ran. (55982 ms total)
[  PASSED  ] 104 tests.
```