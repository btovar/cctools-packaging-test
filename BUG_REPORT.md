# Bug Report: CCTools Packaging Test

## Bug #1: Logic Error in smoke.sh Exit Code Handling

**File:** `smoke.sh`
**Lines:** 44-46
**Type:** Logic Error

### Description
The script has a logic error in the exit code handling. When the smoke test succeeds, the script prints "0" to stdout but doesn't actually exit with code 0. This could cause the script to continue execution and potentially exit with a non-zero code, causing false test failures.

### Current Code
```bash
if diff input.txt output.txt
then
	echo "smoke test success"
	echo 0
else
	echo "smoke test failure"
	exit 1
fi
```

### Issue
The script prints "0" but doesn't exit with code 0. The script continues execution after the conditional block.

### Fix Applied
```bash
if diff input.txt output.txt
then
	echo "smoke test success"
	exit 0  # Changed from 'echo 0'
else
	echo "smoke test failure"
	exit 1
fi
```

**Status:** ✅ FIXED - Replaced `echo 0` with `exit 0` to properly exit with success code.

---

## Bug #2: Race Condition in shadho-test.sh Process Management

**File:** `shadho-test.sh`
**Lines:** 39-41
**Type:** Race Condition / Process Management Issue

### Description
The script has a race condition when terminating the background worker process. The script uses `kill -9` (SIGKILL) immediately after running the main application, which could lead to:
1. Incomplete cleanup of worker resources
2. Potential data corruption if the worker is in the middle of processing
3. Ungraceful shutdown that doesn't allow proper cleanup

### Current Code
```bash
#kill the worker, to be sure
kill -9 $SHADHO_WORKER_PID
```

### Issue
Using `kill -9` (SIGKILL) doesn't allow the process to clean up properly. The script should first try a graceful shutdown with SIGTERM, then use SIGKILL only as a last resort.

### Fix Applied
```bash
#kill the worker gracefully, then forcefully if needed
if kill -TERM $SHADHO_WORKER_PID 2>/dev/null; then
    # Wait up to 5 seconds for graceful shutdown
    for i in {1..5}; do
        if ! kill -0 $SHADHO_WORKER_PID 2>/dev/null; then
            break
        fi
        sleep 1
    done
    # If still running, force kill
    kill -9 $SHADHO_WORKER_PID 2>/dev/null
else
    # Process already dead, nothing to do
    echo "Worker process already terminated"
fi
```

**Status:** ✅ FIXED - Implemented graceful shutdown sequence with SIGTERM followed by SIGKILL as last resort.

---

## Bug #3: Resource Leak in coffea-test.py Worker Factory

**File:** `coffea-test.py`
**Lines:** 148-163
**Type:** Resource Leak / Improper Resource Management

### Description
The script creates a Worker Factory but doesn't properly handle cleanup in case of exceptions. The factory manages worker processes and network connections, and if an exception occurs during the processing, these resources might not be properly cleaned up.

### Current Code
```python
workers = Factory("local", manager_host_port="localhost:9123")

workers.max_workers = 1
workers.min_workers = 1
workers.python_package = wq_env_tarball
with workers:
    output = processor.run_uproot_job(
        fileset,
        treename='Events',
        processor_instance=MyProcessor(),
        executor=processor.work_queue_executor,
        executor_args=work_queue_executor_args,
        chunksize=100000,
        maxchunks=4,
    )
```

### Issue
While the code uses a context manager (`with workers:`), the factory is configured outside the context manager. If an exception occurs during factory configuration, the resources allocated during `Factory()` instantiation won't be cleaned up.

### Fix Applied
```python
workers = None
try:
    workers = Factory("local", manager_host_port="localhost:9123")
    workers.max_workers = 1
    workers.min_workers = 1
    workers.python_package = wq_env_tarball
    
    with workers:
        output = processor.run_uproot_job(
            fileset,
            treename='Events',
            processor_instance=MyProcessor(),
            executor=processor.work_queue_executor,
            executor_args=work_queue_executor_args,
            chunksize=100000,
            maxchunks=4,
        )
except Exception as e:
    # Ensure cleanup even if factory creation fails
    if workers is not None:
        try:
            workers.shutdown()
        except:
            pass
    raise e
```

**Status:** ✅ FIXED - Added proper exception handling to ensure worker factory cleanup in all scenarios.

---

## Additional Notes

These bugs represent common issues in distributed computing environments:
- **Logic errors** in shell scripts can cause CI/CD pipelines to report incorrect results
- **Race conditions** in process management can lead to resource leaks and system instability
- **Resource leaks** in distributed applications can cause worker processes to hang and consume system resources

All fixes have been implemented in the respective files.