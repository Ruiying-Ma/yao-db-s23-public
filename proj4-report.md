# Porject 4: Concurrency Control & Recovery
## Latches

In `transaction/txn.hpp`
```c++
class Txn {
    std::shared_mutex rw_latch_
}
```

In `transaction/txn_manager.hpp`
```c++
class TxnManager {
 public:
  // txn_table holds the memory ownership of all txns.
  static std::unordered_map<txn_id_t, std::unique_ptr<Txn>> txn_table_;
  // latch to protect concurrent access to txn_table.
  static std::shared_mutex rw_latch_;
}
```

In `transaction/lock_manager.hpp`
```c++
  // table-level lock table
  std::unordered_map<std::string /*table_name*/,
  std::unique_ptr<LockRequestList>>
      table_lock_table_;
  std::mutex table_lock_table_latch_;

  // row-level lock table
  std::unordered_map<std::string /*table_name*/,
  std::unordered_map<std::string /*key*/, std::unique_ptr<LockRequestList>>>
      tuple_lock_table_;
  std::mutex tuple_lock_table_latch_;

  class LockRequestList {
      std::list<std::shared_ptr<LockRequest>> list_;
      std::mutex latch_;
      std::condition_variable cv_;
  }
  
```

I also add a recursive latch in the B+ tree: in `bplus-tree.hpp`: 

```c++
class BPlusTree {
private: 
    std::recursive_mutex latch_;
}
```

## Strict 2-PL
- `AcquireTableLock` in `lock_manager.cpp`

    1. Check the txn's state. If `ABORTED` or `SHRINKING`, throw `TxnInvalidBehaviorException`
    2. Lock `table_lock_table_`

        2.1 Find the LockRequestList `table_lock_request_list`

        2.2 Lock `table_lock_request_list`
    3. Unlock `table_lock_table` (after `table_lock_request_list` is locked)
    4. In `table_lock_request_list`: 

        4.1 If the txn already has exactly the same lock it is now acquiring, return

        4.2 If the txn doesn't have this lock

        - If not update lock: append the ungranted request to `table_lock_request_list`
        - If update lock: check whether the update is valid; set `table_lock_request_list->upgrading_` to be the current txn

        4.3 Try to acquire the lock

        ```c++
        bool need_to_wait = false;
        while (true) {
            if (conflict) {
                need_to_wait = true;
                // Handle deadlock using wait-die
                if (wait-die) {
                    set txn->state_ to ABORTED
                    Remove the ungranted request from `table_lock_request_list`
                    Remove the upgrading txn if it is the current txn
                    throw TxnDLAbortException
                } 
            }
            if (some other txn is waiting for upgrading) {
                need_to_wait = true;
            }
            if (some other txns are in front of me in the request list, and they can acquire the lock, and after they acquire the lock, they conflict with the current txn) {
                // FIFO
                need_to_wait = true;
            }
            if (!need_to_wait) {break;}
            else {wait on table_lock_request_list->cv_}
        }
        // Get the lock
        ```
        4.4 Get the lock

        Update the table_lock_request_list and the current txn's table_lock_set_
- `ReleaseTableLock` in `lock_manager.cpp`

    1. Set the txn's state to `SHRINKING`. If the txn's state is `ABORTED`, do nothing.
    2. Lock `table_lock_table_`
    3. Update `table_lock_request_list` (and `table_lock_table_` if necessaty)
    4. Unlock `table_lock_table_`
    5. Update the txn's `table_lock_set_`
- `AcquireTupleLock` in `lock_manager.cpp`

    Almost the same as `AcquireTableLock`, except that we need to check whether 
    1. the target lock is not an intention lock

    2. the corresponding table has already been granted a correct intention lock
- `ReleaseTableLock` in `lock_manager.cpp`

    Almost the same as `ReleaseTableLock`

### How to check whether two `LockModes` are conflicting
When a txn is trying to acquire a lock, it should first check wether there are granted conflicting locks held by other txns. 

||IS|IX|SIX|S|X|
|:-:|:-:|:-:|:-:|:-:|:-:|
|IS| Y | Y | Y | Y | N |
|IX | Y | Y | N | N | N |
|SIX| Y | N | N | N | N |
|S|Y | N | N | Y | N | 
|X | N | N | N | N | N |

## Acquiring locks when accessing the tables/tuples
When a query is executed, it need to access tables and tuples. We need to apply the strict 2-PL locks implemented before to realize concurrent control.

- Acquiring table locks

    In `db.cpp`
    - `GetIterator`

    | Already has lock | Need to acquire lock | 
    |:-:|:-:|
    | IS | S|
    |IX | SIX|
    |SIX | SIX|
    | S| S|
    |X | X|
    | No lock | S|
    - `GetRangeIterator`

    | Already has lock | Need to acquire lock | 
    |:-:|:-:|
    | IS | S|
    |IX | SIX|
    |SIX | SIX|
    | S| S|
    |X | X|
    | No lock | S|
    - `GetModifyHandle`

    | Already has lock | Need to acquire lock | 
    |:-:|:-:|
    | IS | IX|
    |IX | IX|
    |SIX | SIX|
    | S| SIX|
    |X | X|
    | No lock | IX|
    - `GetSearchHandle`

    | Already has lock | Need to acquire lock | 
    |:-:|:-:|
    | IS | IS|
    |IX | IX|
    |SIX | SIX|
    | S| S|
    |X | X|
    | No lock | IS|
- Acquiring tuple locks 

    In `bplus-tree-storage.hpp`

    - `Delete`

        Acquire `X` tuple lock
    - `Insert`

        Acquire `X` tuple lock
    - `Update`

        Acquire `X` tuple lock
    - `Search`

        Acquire `S` tuple lock
## Rollback
In `txn_manager.cpp`, `Abort(Txn* txn)` function.

1. Pop the stack `txn->modify_records_` to roll back the operations.

2. Set the txn's state to `ABORTED`.

3. Release all the txn's locks.

Remark: 
1. Rollback must happens before setting the txn's state to `ABORTED`. Since when rolling back changes, the txn still need to call the functions in `lock_manager.cpp`, while `AcquireTableLock` and `AcquireTupleLock` don't accept aborted txn.
2. When rolling back, each time after the txn rolls back an operation, it need to pop the `modify_records_` stack, since the roll back operation should not be recorded in the stack.

## Results
```
TxnBenchmark.BenchmarkTable
Total commit cnt: 2664128   // std: 2819003
Total abort cnt: 25531

TxnBenchmark.BenchmarkTuple
Total commit cnt: 2374656   // std: 2583597
Total abort cnt: 24868
```

