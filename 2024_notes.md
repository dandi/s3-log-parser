# Notes from 2024 runs

A few summary facts as of 2024:

- A single line of a raw S3 log file can be between 400-1000+ bytes.
- Some of the busiest daily logs on the archive can have around 5,014,386 lines.
- There are more than 6 TB of log files collected in total.
- This parser reduces that total to less than 25 GB of final essential information on NWB assets (Zarr size TBD).





## Workflow

### 1. **Reduction**

In the summer of 2024, this reduced 6 TB of raw logs to less than 170 GB.

The process is designed to be easily parallelized and interruptible, meaning that you can feel free to kill any processes while they are running and restart later without losing most progress.

### 2. **Binning**

In the summer of 2024, this brought 170 GB of reduced logs down to less than 80 GB (20 GB of `blobs` spread across 253,676 files and 60 GB of `zarr` spread across 4,775 files).

### 3. **Mapping to Dandisets**

In the summer of 2024, this brought 80 GB of binned logs down to around 20 GB of Dandiset logs.



## Usage

### Reduction

On Drogon:

```bash
reduce_all_dandi_raw_s3_logs \
  --raw_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs \
  --reduced_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-reduced \
  --maximum_number_of_workers 3 \
  --maximum_buffer_size_in_mb 3000 \
  --excluded_ips < Drogons IP >
```

In the summer of 2024, this process took less than 10 hours to process all 6 TB of raw log data (using 3 workers at 3 GB buffer size).

### Binning

On Drogon:

```bash
bin_all_reduced_s3_logs_by_object_key \
  --reduced_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-reduced \
  --binned_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-binned
```

This process is not as friendly to random interruption as the reduction step is. If corruption is detected, the target binning folder will have to be cleaned before re-attempting.

The `--file_processing_limit < integer >` flag can be used to limit the number of files processed in a single run, which can be useful for breaking the process up into smaller pieces, such as:

```bash
bin_all_reduced_s3_logs_by_object_key \
  --reduced_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-reduced \
  --binned_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-binned \
```

In the summer of 2024, this process took less than 5 hours to bin all 170 GB of reduced logs into the 80 GB of data per object key.

### Mapping

#### Required Environment Variables

The `map_binned_s3_logs_to_dandisets` command requires two environment variables to be set:

1. **IPINFO_CREDENTIALS**: An access token for the ipinfo.io service
  - We use this service to extract general geographic region information (not exact physical addresses) for anonymized geographic statistics
  - We extract country/region information (e.g. "US/California"), while also specially categorizing requests from known services (GitHub, AWS, GCP, VPN).
2. **IP_HASH_SALT**: A salt value for hashing IP addresses
  - We use hashing to anonymize IP addresses in the logs while still allowing for unique identification
- The hashed values are used as keys in our caching system to track regions without storing actual IP addresses

To set `IPINFO_CREDENTIALS`:
1. Register at [ipinfo.io](https://ipinfo.io/) to get an API access token
2. After registration, obtain your access token from your account dashboard
3. Set the `IPINFO_CREDENTIALS` environment variable to this value

```bash
export IPINFO_CREDENTIALS="your_token_here"
```

To set `IP_HASH_SALT`:
1. Use the built-in `get_hash_salt` function (requires access to the original raw log files)

```python
from dandi_s3_log_parser.testing._helpers import get_hash_salt

# Path to the folder containing the raw log files
raw_logs_path = "/path/to/raw/logs"
salt = get_hash_salt(base_raw_s3_log_folder_path=raw_logs_path)
print(f"Generated IP_HASH_SALT: {salt}")
```
2. Set the `IP_HASH_SALT` environment variable to this generated value

```bash
export IP_HASH_SALT="hash_salt_here"
```

On Drogon:

```bash
map_binned_s3_logs_to_dandisets \
  --binned_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-binned \
  --mapped_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-mapped \
  --excluded_dandisets 000108
```

In the summer of 2024, this blobs process took less than 8 hours to complete (with caches; 10 hours without caches) with one worker.

Some Dandisets may take disproportionately longer than others to process. For this reason, the command also accepts `--excluded_dandisets` and `--restrict_to_dandisets`.

This is strongly suggested for skipping `000108` in the main run and processing it separately (possibly on a different CRON cycle altogether).

```bash
map_binned_s3_logs_to_dandisets \
  --binned_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-binned \
  --mapped_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-mapped \
  --restrict_to_dandisets 000108
```

In the summer of 2024, this took less than 15 hours to complete.

The mapping process can theoretically be designed to work in parallel (and thus much faster), but this would take some effort to design. If interested, please open an issue to request this feature.

To then generate summaries and totals (for ease and efficiency of frontend reporting tools):

```bash
generate_all_dandiset_totals --mapped_s3_logs_folder_path < mapped Dandiset logs folder >
generate_archive_summaries --mapped_s3_logs_folder_path < mapped Dandiset logs folder >
update_region_codes_to_coordinates --mapped_s3_logs_folder_path < mapped Dandiset logs folder >
```
