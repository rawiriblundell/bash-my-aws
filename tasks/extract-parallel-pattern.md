# Extract Parallel Execution Pattern for bash-my-aws

## Purpose

Extract the parallel execution pattern found in bash-my-aws into a reusable function that can be easily applied to other functions that need to process multiple items concurrently.

## Current Pattern Location

The pattern is currently implemented in `lib/s3-functions` starting at line 255:

```bash
# Launch parallel requests for each storage type
for storage_type in "${storage_types[@]}"; do
  (
    # Command to run in parallel
    aws cloudwatch get-metric-statistics \
      --namespace AWS/S3 \
      --metric-name BucketSizeBytes \
      --dimensions Name=BucketName,Value="$bucket_name" Name=StorageType,Value="$storage_type" \
      --start-time "$start_time" \
      --end-time "$end_time" \
      --period 86400 \
      --statistics Average \
      --output json >"${tmp_dir}/${bucket_name}/${storage_type}.json"
  ) &

  # Limit the number of parallel processes to avoid overwhelming the system
  # Adjust this number based on your system's capabilities
  if [[ $(jobs -r | wc -l) -ge 10 ]]; then
    wait -n
  fi
done

# Wait for all background jobs to complete for this bucket
wait
```

## Key Components

1. **Background execution**: Using `&` to run commands in background
2. **Job limiting**: `jobs -r | wc -l` to count running jobs
3. **Throttling**: `wait -n` to wait for next job when limit reached
4. **Completion**: Final `wait` to ensure all jobs finish
5. **POSIX compliance**: No dependency on GNU parallel or xargs

## Proposed Implementation

Create a reusable function like:

```bash
# Run commands in parallel with job limit
# Usage: parallel_execute <max_jobs> <command> <args...>
parallel_execute() {
  local max_jobs="${1:-10}"
  shift
  
  # Execute command in background
  ("$@") &
  
  # Limit concurrent jobs
  if [[ $(jobs -r | wc -l) -ge $max_jobs ]]; then
    wait -n
  fi
}

# Usage example:
for item in "${items[@]}"; do
  parallel_execute 10 process_item "$item"
done
wait  # Wait for all to complete
```

## Functions That Could Benefit

1. **stack-drift-detect** - Process multiple stacks concurrently
2. **bucket-inventory-configuration** - Check multiple buckets
3. **instance-ssm-associations** - Query multiple instances
4. Any function processing lists from skim-stdin

## Benefits

- **Performance**: Significant speedup for operations across multiple resources
- **Scalability**: Handles hundreds of resources efficiently
- **Portability**: Works on any POSIX shell without external dependencies
- **Control**: Configurable concurrency limit prevents overwhelming systems

## Next Steps

1. Extract pattern into a helper function
2. Test with various bash-my-aws functions
3. Document usage in CONVENTIONS.md
4. Update existing functions to use the pattern where beneficial