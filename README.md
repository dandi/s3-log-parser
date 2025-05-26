<p align="center">
  <h1 align="center">S3 Log Parser</h3>
  <p align="center">
    <a href="https://pypi.org/project/dandi_s3_log_parser/"><img alt="Ubuntu" src="https://img.shields.io/badge/Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white"></a>
    <a href="https://pypi.org/project/dandi_s3_log_parser/"><img alt="Supported Python versions" src="https://img.shields.io/pypi/pyversions/dandi_s3_log_parser.svg"></a>
    <a href="https://codecov.io/github/CatalystNeuro/dandi_s3_log_parser?branch=main"><img alt="codecov" src="https://codecov.io/github/CatalystNeuro/dandi_s3_log_parser/coverage.svg?branch=main"></a>
  </p>
  <p align="center">
    <a href="https://pypi.org/project/dandi_s3_log_parser/"><img alt="PyPI latest release version" src="https://badge.fury.io/py/dandi_s3_log_parser.svg?id=py&kill_cache=1"></a>
    <a href="https://github.com/dandi/s3-log-parser/blob/main/license.txt"><img alt="License: BSD-3" src="https://img.shields.io/pypi/l/dandi_s3_log_parser.svg"></a>
  </p>
  <p align="center">
    <a href="https://github.com/psf/black"><img alt="Python code style: Black" src="https://img.shields.io/badge/python_code_style-black-000000.svg"></a>
    <a href="https://github.com/astral-sh/ruff"><img alt="Python code style: Ruff" src="https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json"></a>
  </p>
</p>

This project was an attempt at rigorous parsing of full information from raw S3 logs.

This remains an unsolved problem, with others having attempted in the past:
  - [joswr1ight: s3logparse](https://github.com/joswr1ght/s3logparse/tree/main)
    - Unmaintained for 4 years.
    - Minimally tested: https://github.com/joswr1ght/s3logparse/blob/4df6a40e11420c132420336f09ef4604c67cc171/tests/s3logparse_test.py
    - Biggest weakness is it [reads the entire log file at once](https://github.com/joswr1ght/s3logparse/blob/4df6a40e11420c132420336f09ef4604c67cc171/s3logparse.py#L144), which is sufficient to crash most systems due to some of our files being larger than most RAM chips available.
  - [cocoonlife: s3-log-parse](https://github.com/cocoonlife/s3-log-parse) (imported as `s3logparse`, just to add confusion with above)
    - Unmaintained for 6 years.
    - Used for a while in the built-in DANDI archive 'download counter'.
    -  Untested and unvalidated.
    -  Difficult to install in modern environments: https://github.com/cocoonlife/s3-log-parse/issues/4
    - Suffers from some of the same parsing errors we encounter on our worst URIs: https://github.com/cocoonlife/s3-log-parse/issues/1
    -  Also [reads the entire log file at once](https://github.com/cocoonlife/s3-log-parse/blob/c19954bde2913b439c20d8f8bb3a22c9490e4b62/s3logparse/cli.py#L21), which is sufficient to crash most systems due to some of our files being larger than most RAM chips available.

This work has transitioned to [s3-log-extraction](https://github.com/dandi/s3-log-extraction), which instead focuses on the development of efficient heuristics that extract only the minimal fields we desire for reporting summary activity with the public.

As such, this repository will be left open to allow others to request its revival by opening an issue.

Developed for the [DANDI Archive](https://dandiarchive.org/).

Read more about [S3 logging on AWS](https://web.archive.org/web/20240807191829/https://docs.aws.amazon.com/AmazonS3/latest/userguide/LogFormat.html).



## Installation

```bash
pip install dandi_s3_log_parser
```



## Workflow

The process is comprised of three modular steps.

### 1. **Reduction**

Filter out:

- Non-success status codes.
- Excluded IP addresses.
- Operation types other than the one specified (`REST.GET.OBJECT` by default).

Then, only limit data extraction to a handful of specified fields from each full line of the raw logs; by default, `object_key`, `timestamp`, `ip_address`, and `bytes_sent`.

The process is designed to be easily parallelized and interruptible, meaning that you can feel free to kill any processes while they are running and restart later without losing most progress.

### 2. **Binning**

To make the mapping to Dandisets more efficient, the reduced logs are binned by their object keys (asset blob IDs) for fast lookup. Zarr assets specifically group by the parent blob ID, *e.g.*, a request for `zarr/abcdefg/group1/dataset1/0` will be binned by `zarr/abcdefg`.

This step reduces the total file sizes from step (1) even further by reducing repeated object keys, though it does create a large number of small files.

### 3. **Mapping**

The final step, which should be run periodically to keep the desired usage logs per Dandiset up to date, is to scan through all currently known Dandisets and their versions, mapping the asset blob IDs to their filenames and generating the most recently parsed usage logs that can be shared publicly.



## Usage

### Reduction

To reduce:

```bash
reduce_all_dandi_raw_s3_logs \
  --raw_s3_logs_folder_path < base raw S3 logs folder > \
  --reduced_s3_logs_folder_path < reduced S3 logs folder path > \
  --maximum_number_of_workers < number of workers to use > \
  --maximum_buffer_size_in_mb < approximate amount of RAM to use > \
  --excluded_ips < comma-separated list of known IPs to exclude >
```

### Binning

To bin:

```bash
bin_all_reduced_s3_logs_by_object_key \
  --reduced_s3_logs_folder_path < reduced S3 logs folder path > \
  --binned_s3_logs_folder_path < binned S3 logs folder path >
```

This process is not as friendly to random interruption as the reduction step is. If corruption is detected, the target binning folder will have to be cleaned before re-attempting.

The `--file_processing_limit < integer >` flag can be used to limit the number of files processed in a single run, which can be useful for breaking the process up into smaller pieces, such as:

```bash
bin_all_reduced_s3_logs_by_object_key \
  --reduced_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-reduced \
  --binned_s3_logs_folder_path /mnt/backup/dandi/dandiarchive-logs-binned \
```

### Mapping to Dandisets

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

To map:

```bash
map_binned_s3_logs_to_dandisets \
  --binned_s3_logs_folder_path < binned S3 logs folder path > \
  --mapped_s3_logs_folder_path < mapped Dandiset logs folder > \
  --excluded_dandisets < comma-separated list of six-digit IDs to exclude > \
  --restrict_to_dandisets < comma-separated list of six-digit IDs to restrict mapping to >
```


## Submit line decoding errors

Please email line decoding errors collected from your local config file (located in `~/.dandi_s3_log_parser/errors`) to the core maintainer before raising issues or submitting PRs contributing them as examples, to more easily correct any aspects that might require anonymization.
