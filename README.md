# Turingarena CommonBox API

A minimal API for a sandbox, supporting writing input, reading output, measuring and limiting resource usage.

## Goals

* Allow implementations where features are only partially supported.
* Support I/O streaming on two channels (usually stdin/stdout).
* Support both batch and streaming I/O, if provided by the underlying sandbox.
* Support detailed measurement of resource usage (cumulative CPU time, current and peak memory usage, during and after the execution), if provided by the underlying sandbox.
* Simplifying the handling of error conditions (illegal operations, timeouts, exceeded resource limits, etc.) by API users.

## Non-goal

The following feature can be obtained with sandbox-specific APIs, but they are not covered by this API.

* Fine-grained control on permissions (e.g., reading/writing files, spawning threads or processes, etc.).
* Multiple input or output files or streams.

## API

### Creating and closing processes

```c
typedef void* commonbox_process_t;

commonbox_process_t commonbox_create_process_from_executable(char* path);
```

Creates a new sandboxed process from a given executable file.

```c
uint64_t commonbox_close(commonbox_process_t process);
```

Closes a process.
This method must be called exactly once, and it releases any resource used by this process.
After this method is called, the process cannot be used for any other call.

### Error handling

```c
typedef bool result_t;
```

Method that can fail return the above type (TODO: currently defined as `bool`, but could be a more involved definition).

All the following methods MUST NOT fail due to the state of the sandboxed process.
In other words, if a sequence of calls succeeds with a given sandboxed process, it MUST succeed with any other sandboxed process.
This is especially important with method for writing the input (see below).

### Writing the input

```c
result_t commonbox_write_file_at(commonbox_process_t process, char* path);
result_t commonbox_write_file(commonbox_process_t process, int fd, off_t offset, size_t len);
result_t commonbox_write(commonbox_process_t process, char* buf, size_t len);
```

Depending on the method used, a piece of input is specified as either, in order of increasing generality:

* a path to a file,
* a file descriptor with an offset and length, or
* a pointer to a byte buffer with a length.

Each method returns a successful value if the sandbox supports sending the input as specified at the time of the call.

If the sandboxed process is not started when these methods are called, it is started.

*These methods MUST NOT fail for the sole reason that the sandboxed process is in a state in which it cannot receive any more input (such as terminated or crashed). Instead, the input should be simply ignored.
In particular, if the process exits, crashes, or is terminated due to exceeded resources (including wall-clock time)
while the input was being writted, the remaining input is simply ignored.*

The input may be kept in a buffer which is not necessarily flushed when the method returns.

A sandbox MUST support at least a single call to `commonbox_write_file_at` at the beginning, before reading any output.
If a sandbox supports a method with a given generality, then it SHOULD support also methods with lower generality.

### Reading the output

```c
int commonbox_read(commonbox_process_t process, char* buf, size_t len);
```

This method flushes any buffered input to the sandboxed process, then reads at least one and up to `len` bytes from the process output.
If the process will not produce any more output (because it has exited, crashed, or has been terminated due to exceeded resources),
then it returns `-1`.
Otherwise, it returns the number of bytes read, and the bytes are written to `buf`.

If the sandboxed process is not started when this method is called, it is started before attempting to read any output.

This method is (by design) the only way one can know if the sandboxed process is terminated (for any reason).

### Safety limits

```c
result_t commonbox_set_safety_timeout(commonbox_process_t process, uint64_t nanos);
result_t commonbox_set_safety_time_limit(commonbox_process_t process, uint64_t nanos);
result_t commonbox_set_safety_memory_limit(commonbox_process_t process, uint64_t bytes);
result_t commonbox_set_safety_output_limit(commonbox_process_t process, uint64_t bytes);
```

Limit the overall resource usage of the sandbox.
Each of these limits has a reasonable default, if not set explicitly.
These limits exist for safety only, and should not be used to check the resource usage of the sandboxed process for evaluation purposes.
Each method returns a successful value if the sandbox can enforce the specified limit at the time of the call, an error otherwise (which may be ignored).

If supported by the sandbox, the limits can be set during the execution of the program.
In this case, the new limit refers to the resource used *after* the call is performed.
For example, if the safety time limit is repeatedly set to one second during the execution (and the sandbox supports this pattern of usage), then the overall time usage can grow arbitratily large.

### Measuring resource usage

```c
uint64_t commonbox_get_cumulative_time_usage(commonbox_process_t process, bool upper);
uint64_t commonbox_get_current_memory_usage(commonbox_process_t process, bool upper);
uint64_t commonbox_get_peak_memory_usage(commonbox_process_t process, bool upper);
```

Measure the resources used by the sandboxed process.
If `upper` to true, an upper bound is returned, otherwise they return a lower bound on the usage of the specific resource measured.
Implementations may return `0` (if `upper` is false) or `UINT64_MAX` (if `upper` is true) if they cannot produce a reliable measure at the time of the call.
Time is measured in nanoseconds, memory is measured in bytes.

```c
result_t commonbox_reset_peak_memory_usage(commonbox_process_t process);
```

Resets peak memory usage to the current memory usage.
Returns a successful value if supported by the sandbox at the time of call.

### Getting the reason for termination

```c
char* commonbox_get_exit_reason(commonbox_process_t process);
void commonbox_force_exit(commonbox_process_t process);
```

Returns a pointer to a NULL-terminated string describing the reason why the process is terminated.
If the process is still running, it first waits for it to exit, crash, or be terminated due to exceeded resource limits (which will happen eventually).
The returned pointer is only valid until the process is closed.

Use `commonbox_force_exit` before `commonbox_get_exit_reason` to avoid waiting for resource exhaustion.

Implementations may require `commonbox_get_exit_reason` to be called before being able to provide resource usage measurements.

Examples of termination reasons are:

* *Exited normally (return code 0)*.
* *Exited with error (return code 1)*.
* *Exited with signal SIGILL (illegal instruction)*.
* *Terminated (forced exit)*.
* *Terminated (timeout expired)*.
* *Terminated (time limit exceeded)*.
* *Terminated (memory limit exceeded)*.
* *Terminated (output limit exceeded)*.
