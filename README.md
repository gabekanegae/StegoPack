# StegoPack

Final project for **[SCC0251 - Image Processing](https://uspdigital.usp.br/jupiterweb/jupDisciplina?sgldis=SCC0251)** @ ICMC/USP.

`StegoPack` is a Python module and full application that is able to encode any file into an image via **LSB steganography**, as well as detect and decode a file from an image. For that, the lowest level of encoding is be selected (the one that degrades the original image the least), based on file sizes.

This implementation focuses on modularity/flexibility (**Module Usage** below) and speed (multithreading implementation). The Python Notebooks also provide a more visual exploration of the method and its results.

The encoding levels available are:

* **L0**: 1-bit LSB
* **L1**: 2-bit LSB
* **L2**: 4-bit LSB

## Dependencies

- [`NumPy 1.17.4`](https://numpy.org/) module. Install with `pip3 install numpy`.
- [`imageio 2.8.0`](https://pypi.org/project/imageio/) module. Install with `pip3 install imageio`.

## Files

* [`StegoPack.py`](StegoPack.py) is the main module, containing the `Image` and `Payload` classes. Can also be run standalone as a full application via CLI.
* [`Demo.ipynb`](Demo.ipynb) is a Jupyter Notebook containing a few examples using [`StegoPack.py`](StegoPack.py) and files from [`demo_files/`](demo_files/).
* [`Analysis.ipynb`](Analysis.ipynb) is a Jupyter Notebook with a deeper analysis and discussion over the results.
* [`regression_testing_and_benchmark.py`](regression_testing_and_benchmark.py) is a small helper script to run some examples, used on development to check for regression bugs. Also doubles as a benchmark, to evaluate improvements on runtime.

## Standalone Application Usage

* `python3 StegoPack.py`
  * Show usage info.

* `python3 StegoPack.py (imageFilename)`
  * Show image info and storage capabilities. Detects and decodes any payload in it.

* `python3 StegoPack.py (imageFilename) (payloadFilename) (outputFilename)`
  * Encode `payloadFilename` into `imageFilename` and output as `outputFilename`.

## Module Usage (Quick Start)

All you need to use this module on your projects are the requirements above and [`StegoPack.py`](StegoPack.py) itself. Here's some basic boilerplate:

```python3
from StegoPack import *

# Encoding
image = Image(imageFilename)
payload = Payload(payloadFilename)
image.encodePayload(payload)
image.saveFile(encodedImageFilename)

# Decoding
image = Image(encodedImageFilename)
payload = image.decodePayload()
payload.saveFile()
```

## Implementation Details

### LSB Steganography

LSB steganography consists in replacing the last N bits of every subpixel in the image with the appropriate bits of the data. As `N = {1, 2, 4}`, all bytes of the data can be split in a set number of subpixels.

The encoding is done at `Image.encodePayload()`: the payload header is assembled (`Payload.getBytes()`), the encoding level is determined based on the total payload size and image resolution, and the data is converted to a 3D matrix with the same shape as the image. Finally, the last N bits of all subpixels are replaced with the payload.

The decoding, at `Image.decodePayload()`, starts with a scan for a payload (and its encoding level) by `Image.hasPayload()` - if none is detected, a `ValueError` exception is raised. If one is found, the payload header is read and assembled, allowing the data to be read. The data bytes are extracted from the image via `Image.__readNextNBytes()`, which in turn calls `Image._readNBytes()` - single-threaded if the amount of bytes is small, multi-threaded otherwise. In the end, a checksum of the data read is calculated and checked against the one from the header, throwing an `AssertionError` if they don't match.

A more detailed breakdown of both the payload header and the parallelization methods are below.

### Payload Header

When the payload is encoded/decoded, a header is placed in the base file preceding the actual data, storing metadata. Here is a more detailed look at its format and fields:

|`encoding`|`level`|`filename-size`|`filename`|`data-hash`|`data-size`|
|-|-|-|-|-|-|
|1 byte|1 byte|1 byte|`filename-size` bytes|32 bytes|4 bytes|

* `encoding`: always 0.
* `level`: can be 0, 1 or 2. Determines the encoding level (1-bit LSB, 2-bit LSB, 4-bit LSB, respectively).
* `filename-size`: length of `filename` field.
* `filename`: payload original filename.
* `data-hash`: SHA-256 hash of the payload data, to be checked against the decoded data.
* `data-size`: payload data size.

After the header, a sequence of `data-size` bytes follows, containing the actual payload data.

### Parallelization/Vectorization

In theory, both encoding and decoding can be parallelized. However, as encoding consists mainly of writing to a big matrix (the resulting image) and Python's `multiprocessing` library is not fond of shared memory, that proved to be quite a challenge. Instead, encoding has been dramatically sped up by making heavy use of NumPy's vectorization methods, at the end of `Image.encodePayload()`.

For decoding, a proper parallelization method was applied with `multiprocessing.Pool` using `threads` threads: each thread `t` decodes `n` payload bytes with indexes in `range(s, e)`, with `s = t * (n//threads) + min(n%threads, t)` and `e = (t+1) * (n//threads) + min(n%threads, (t+1))`. With encoding bit amounts being powers of 2, every thread is able to process its workload independently without the need to worry about race conditions. Overhead should be minimal as assigned job indexes are determined in O(1) time by the expressions above, thus being contiguous and evenly split between threads. This section of code is present at the `Image.__readNextNBytes()` method.

## Examples

As the application is supposed to work with any image and payload file as input, the examples listed below are random images and files from the internet, with varying resolutions, styles and file sizes as to showcase the different distortions from different encoding levels.

Side-to-side comparisons are available in the [`Demo.ipynb`](Demo.ipynb) and [`Analysis.ipynb`](Analysis.ipynb) Jupyter Notebooks.

Input Image | Input Size | Payload | Payload Size | Encoding | Output Image | Output Size
-|-|-|-|-|-|-|
[corgi-599x799.jpg](demo_files/corgi-599x799.jpg) | 66.2 KB | [faustao.png](demo_files/payloads/faustao.png) | 97.1 KB | L0 | [corgi-L0.png](demo_files/encoded/corgi-L0.png) | 671 KB
[plush-418x386.jpg](demo_files/plush-418x386.jpg) | 16.6 KB | [paperpeople.txt](demo_files/payloads/paperpeople.txt) | 3.3 KB | L0 | [plush-L0.png](demo_files/encoded/plush-L0.png) | 156 KB
[caliadventure-1080x1350.jpg](demo_files/caliadventure-1080x1350.jpg) | 201.3 KB | [git-cheatsheet.pdf](demo_files/payloads/git-cheatsheet.pdf) | 352.8 KB | L0 | [caliadventure-L0.png](demo_files/encoded/caliadventure-L0.png) | 2.1 MB
[caliadventure-1080x1350.jpg](demo_files/caliadventure-1080x1350.jpg) | 201.3 KB | [randall.zip](demo_files/payloads/randall.zip) | 1.0 MB | L1 | [caliadventure-L1.png](demo_files/encoded/caliadventure-L1.png) | 2.5 MB
[nightfall-1920x1080.jpg](demo_files/nightfall-1920x1080.jpg) | 1.1 MB | [hap.mp4](demo_files/payloads/hap.mp4) | 1.2 MB | L1 | [nightfall-L1.png](demo_files/encoded/nightfall-L1.png) | 2.8 MB
[gravityfalls-1435x837.jpg](demo_files/gravityfalls-1435x837.jpg) | 325 KB | [hap.mp4](demo_files/payloads/hap.mp4) | 1.2 MB | L2 | [gravityfalls-L2.png](demo_files/encoded/gravityfalls-L2.png) | 2.0 MB
[randall-2560x1372.png](demo_files/randall-2560x1372.png) | 1.2 MB | [pier39.mp4](demo_files/payloads/pier39.mp4) | 3.2 MB | L2 | [randall-L2.png](demo_files/encoded/randall-L2.png) | 4.3 MB
