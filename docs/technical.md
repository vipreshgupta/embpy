# Architecture

![embpy architecture](embpy_architecture.svg)

`embpy` relies on a highly modular architecture that isolates heavy dependencies, ensuring that the core package remains lightweight and accessible.

## The Embedder Protocol

At the heart of `embpy` is the `BioEmbedder`. It acts as a factory, instantly instantiating specific wrappers based on the model requested.

All underlying embedding wrappers inherit from the `BaseModelWrapper` interface:

```python
class BaseModelWrapper:
    def load(self, device: torch.device):
        pass
        
    def embed(self, sequences: list[str]) -> np.ndarray:
        pass
```

### Lazy Imports
Wrappers perform lazy imports of heavy libraries like `transformers` or `torch_geometric` inside their `load()` methods. This allows users to import `embpy` in fractions of a second, without forcing the entire deep learning stack into memory.

## Adding a New Model

To add a new model to `embpy`:
1. Create a class inheriting from `BaseModelWrapper`.
2. Implement `load()` and `embed()`.
3. Register the class in `embpy/models/__init__.py`.
