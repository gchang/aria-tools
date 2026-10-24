# Changes from upstream

Upstream: https://github.com/aria-tools/ARIA-tools, tag `v1.6.1`, commit 74358b5f694002519900cf3a1532ad2fadb7b5ed (2026-04-01), by David Bekaert, Simran Sangha and the ARIA team at JPL/Caltech (see `CITATION`), Apache License 2.0. Development was funded by NASA ROSES (Sea-level Change Team, Earth Surface and Interior) and the NISAR Science Team.

This copy starts from upstream's file tree at 74358b5 without its git history. It is unchanged except for the items below; the LICENSE and CITATION are untouched. `.github/` (the CI workflow and the issue templates) was left out.

Upstream's `main` has two newer commits as of 2026-10-24 (8e59ff2 and 57b6917). Where they fix the same problem, the change below is a backport and names the commit.

## 2026-10-24: extraction works with GDAL 3.13

`tools/ARIAtools/util/meta_cache.py` opened `NETCDF:"<file>` without the closing quote (in `extract_metadata_gdal` and `get_h5_field`). GDAL 3.12 accepted that; GDAL 3.13, which `environment.yml` (`gdal>=3.7.0`) resolves to today, does not, so every `ariaExtract.py` and `ariaTSsetup.py` run stopped with `RuntimeError: NETCDF:"<file>: No such file or directory`. Both strings now end with `"`. This is the same two-line change as upstream commit 57b6917. Results do not change: GDAL 3.13 with the fix gives the same outputs as GDAL 3.12 without it.

## 2026-10-24: `ariaDownload.py --version` keeps the matching scenes

The version filter returned a shorter URL list, and the scenes and pair names were then matched to it by position, so the scenes kept were not the ones whose URLs passed the filter. With `-t 124 --ifg 20180420_20180315 --version 2_0_6`, no product was selected although two v2_0_6 products match. Scenes, pair names and URLs are now filtered against the full URL list. This is the Sentinel-1 part of upstream commit 57b6917; the NISAR version option that commit also adds is not included (`--version` with NISAR products still stops with an error, as before). Runs without `--version` do not change.

## 2026-10-24: `ariaDownload.py` downloads each file once

The HTTPS retry loop in `download_scenes` had no exit after a good download, so it kept going until `max_retries` (3) and fetched every product three times, writing over the same file. It now returns after a complete download, as the S3 branch already did. The files are the same; only the extra traffic and time go away.

## 2026-10-24: `ariaDownload.py` finds Sentinel-1 products with asf_search 13 and newer

The Sentinel-1 search passed `collections=["C2859376221-ASF", "C1261881077-ASF"]` together with `processingLevel=GUNW_STD`. Since asf_search 13.0.0, `GUNW_STD` maps to the SENTINEL-1_INTERFEROGRAMS collection (the v2 products), while those two collection IDs hold the v3 products (ARIA_S1_GUNW), so the two filters no longer overlap. Every search returned `Found -- 0 -- products` with exit code 0. `environment.yml` asks for `asf_search>=12.0.3`, which resolves to 14.x today.

The `ARIA S1 GUNW` dataset already covers both collections, so the search now uses the dataset alone, as upstream commit 57b6917 did for the NISAR search. With asf_search 14.1.0, track 124 from 2018-03-14 to 2018-04-21 returns the same 76 products (v2_0_1, v2_0_5, v2_0_6 and v3_0_1) that asf_search 12 returned with the old arguments, and `--ifg 20180420_20180315` keeps 4 of them. The NISAR search was checked and still finds products with asf_search 14.1.0; it is unchanged.

## 2026-10-24: MintPy listed in `environment.yml` and `requirements.txt`

The last working step of `tests/test.sh` loads the `ariaTSsetup.py` stack into MintPy with `prep_aria.py`, but MintPy was not listed, so that step failed with "command not found". ARIA-tools itself does not import MintPy.

## Not changed

- `tests/test.sh` still ends with `ariaExtract.py -m 'NLCD'`, which fails because NLCD is no longer a mask option (`Could not open user mask: .../NLCD`). It runs after all the other steps.
- `tests/test.sh` still fetches the two test products with `wget` rather than `ariaDownload.py`.
- `setup.py` still imports `distutils`, which works on Python 3.13 through the shim in setuptools.
- `Product already read` in `tools/ARIAtools/product.py` logs the wrong file name (upstream fixed this in 8e59ff2); it only affects a log message.

## Version

`setup.py` takes the version from `git describe --tags` (the part before the first `-`), so it comes from the git tag on the commit you install from. Without a tag, or from a source archive without `.git`, the package installs as version `0`. Tag names must not contain `-` (for example `v1.6.1.post1`, not `v1.6.1-post1`).

## Check

On 2026-10-24, on macOS with 2 threads: a new conda environment built from `environment.yml` (Python 3.13.17, GDAL 3.13.3, asf_search 14.1.0, MintPy 1.6.4), then `python -m pip install -e .` from a fresh clone, as in the README.

- `pytest tests/test_height_subset.py`: 70 passed, 10 skipped (the skipped tests need a real GUNW file).
- `tests/test.sh`, step by step: `ariaDownload.py -o count` reports 4 products; the two v2_0_6 test products (md5 `7f7f5a4bdcbf147dccf067af86106d83`, `6482a590ec3c21a897d22e5eb53bbd0a`) download with `wget`; the three `ariaExtract.py` runs, `ariaTSsetup.py` and `prep_aria.py` all succeed; the NLCD step fails as noted above.
- The outputs match the earlier GDAL 3.12 run of the unmodified v1.6.1: every valid coherence pixel equals a source GUNW pixel, the unwrapped phase equals the source or the source plus the stitching offset (6.3458 rad for the south frame), with the same pixel counts.
- `ariaDownload.py -t 124 -w products --ifg 20180420_20180315 -nt 2` downloads 4 products, one request each (three each before the fix), and the v2_0_6 files are identical to the `wget` copies. `--version 2_0_6 -o url` lists the two v2_0_6 products.
- Peak memory was 8.8 GB for `ariaTSsetup.py` and 4.9 GB for `ariaExtract.py -l azimuthAngle -d download`; the whole test took about 4.5 minutes, plus about 1 minute to download the 4 products.
