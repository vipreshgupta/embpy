# Getting Started

## Installation

`embpy` supports Python ≥ 3.11 on Linux, macOS, and Windows.

=== "Linux / macOS"

    ```bash
    # CPU-only (Note: On Linux pip, install torch first to avoid CUDA drivers)
    pip install torch --index-url https://download.pytorch.org/whl/cpu
    pip install "embpy[cpu]"

    # GPU (NVIDIA CUDA)
    pip install "embpy[gpu]"
    ```

=== "Windows"

    ```powershell
    # CPU-only
    py -3.12 -m pip install "embpy[cpu]"

    # GPU (NVIDIA CUDA)
    py -3.12 -m pip install "embpy[gpu]"
    ```

## Minimal Working Example

This example demonstrates how to load a model and embed sequences in 4 lines of code.

```python
import embpy
import torch
from embpy.embedder import BioEmbedder

# Automatically selects GPU if available
device = 'cuda' if torch.cuda.is_available() else 'cpu'

# Initialize the embedder
embedder = BioEmbedder(device=device)

# Embed two protein sequences
sequences = ["MKTVRQERLKSIVRILERSKEPVSGAQLAEELSVSRQVIVQDIAYLRSLGYNIVATPRGYVLAGG",
             "KALTARQQEVFDLIRDHISQTGMPPTRAEIAQRLGFRSPNAAEEHLKALARKGVIEIVSGASRGIRLLQEE"]

embeddings = embedder.embed(
    sequences, 
    entity_type="protein",
    model="esm2_8M",
    output="numpy"
)

print(f"Embedding shape: {embeddings.shape}")
```
