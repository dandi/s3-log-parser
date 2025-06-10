# ATTENTION

This work has transitioned to [s3-log-extraction](https://github.com/dandi/s3-log-extraction), which instead focuses on the development of efficient heuristics that extract only the minimal fields we desire for reporting summary activity with the public.



# Summary

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

As such, this repository will be left open to allow others to request its revival by [opening an issue](https://github.com/dandi/s3-log-parser/issues/new).
