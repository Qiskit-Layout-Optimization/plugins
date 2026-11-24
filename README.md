# HOWTO

Every Python dependency is listed in `pyproject.toml`.

The Qiskit plugin links against the Glasgow Subgraph Solver, so the solver must be compiled before installing the plugin.

## Build the Glasgow Subgraph Solver

The default build assumes that `glasgow-subgraph-solver` and the Qiskit repository are sibling directories, for example:

```text
qiskit-glasgow-work/
├── glasgow-subgraph-solver/
└── qiskit/
    └── test_wrapper/
```

Compile Glasgow in release mode:

```bash
cd glasgow-subgraph-solver

cmake --preset release
cmake --build --preset release
```

The resulting static library should be located at:

```text
build/gss/libglasgow_subgraphs.a
```

On macOS, Glasgow requires GMP. If it is not already installed:

```bash
brew install gmp
```

## Build and install the Qiskit plugin

After compiling Glasgow, move to the plugin directory:

```bash
cd ../qiskit/test_wrapper
```

Build and install the plugin:

```bash
python -m pip install . --no-build-isolation --no-deps
```

For development, an editable installation can be used instead:

```bash
python -m pip install -e . --no-build-isolation
```

The build system normally discovers the sibling `glasgow-subgraph-solver` repository automatically.

If the solver is stored elsewhere, the following environment variables can be used to override the default paths:

```bash
export GLASGOW_SOLVER_DIR=/path/to/glasgow-subgraph-solver
export GLASGOW_BUILD_DIR=/path/to/glasgow-subgraph-solver/build
export GLASGOW_LIBRARY=/path/to/glasgow-subgraph-solver/build/gss/libglasgow_subgraphs.a
```

`GLASGOW_LIBRARY` is useful when the static library is not in the default build location.

## Testing

Call `transpile` on an example circuit and select the Glasgow plugins for the layout, routing, and optimization stages:

```python
from qiskit import QuantumCircuit, transpile
from qiskit.providers.fake_provider import GenericBackendV2

edges = [
    (0, 1), (1, 0),
    (1, 2), (2, 1),
    (2, 3), (3, 2),
    (3, 4), (4, 3),
]

backend = GenericBackendV2(
    num_qubits=5,
    coupling_map=edges,
)

qc = QuantumCircuit(3)
qc.h(0)
qc.cx(0, 1)
qc.cx(1, 2)
qc.measure_all()

result = transpile(
    qc,
    backend=backend,
    optimization_level=3,
    layout_method="glasgow",
    routing_method="glasgow",
    optimization_method="glasgow",
    seed_transpiler=42,
)

print(result.draw(output="text"))
print("Final layout:", result.layout)
print("Operations:", result.count_ops())
```

Using all three Glasgow stage plugins replaces Qiskit's VF2-based initial layout search and both VF2PostLayout uses while leaving the remaining preset pass-manager workflow unchanged.
