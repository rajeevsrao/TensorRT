# Inspecting Tactic Replay Files

The `inspect tactics` subtool can display information about TensorRT tactic replay
files generated by Polygraphy.

For example, first we'll generate a tactic replay file:

```bash
polygraphy run identity.onnx --trt --save-tactics replay.json
```

Next, we can inspect it:

```bash
polygraphy inspect tactics replay.json
```

This will display something like:

```
[I] Layer: node_of_y
        Algorithm: (Implementation: -2147483642, Tactic: 0) | Inputs: (('TensorFormat.LINEAR', 'DataType.FLOAT'),) | Outputs: (('TensorFormat.LINEAR', 'DataType.FLOAT'),)
```
