# Cross-Platform & Hardware

`embpy` is designed to run everywhere, but different OS and Hardware combinations have different constraints.

## OS Constraints

### POSIX Package Gating (Windows)
Certain underlying genomics libraries (`pysam`, `pyensembl`) rely heavily on POSIX features and C-compilers that are difficult or impossible to build on Windows. 

`embpy` automatically excludes these from Windows installations. As a result, direct `.bed` or `.bam` sequence extraction APIs are disabled on Windows.

### Windows Multiprocessing Constraints
Windows uses the `spawn` method for multiprocessing rather than the POSIX `fork`. When using `embpy.embedder.BioEmbedder` with high `num_workers`, ensure your main execution block is guarded:

```python
if __name__ == '__main__':
    embedder = BioEmbedder(device='cpu')
    embedder.embed(data)
```
Failure to do this on Windows will result in an infinite recursion loop spawning subprocesses.

## Hardware Configurations

### PyTorch Memory Access Safety
When embedding extremely large sequence batches on GPUs, PyTorch can occasionally run out of CUDA memory (OOM). `embpy` handles this automatically via `batch_size="auto"`, which intercepts memory access violations, scales the batch size down by 50%, and retries the batch transparently.

### Volta (V100) GPU Workarounds
Standard PyTorch `>=2.5.1` pip wheels no longer include `sm_70` (Volta) kernel images. If you are running on an older V100 GPU:
1. Run your code inside an NVIDIA NGC Container (`nvcr.io/nvidia/pytorch:24.02-py3`).
2. Alternatively, compile PyTorch from source.
