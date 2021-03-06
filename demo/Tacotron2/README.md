# Tacotron 2 and WaveGlow Inference with TensorRT

The Tacotron2 and WaveGlow models form a text-to-speech (TTS) system that enables users to synthesize natural sounding speech from raw transcripts without any additional information such as patterns and/or rhythms of speech. This is an implementation of Tacotron2 for PyTorch, tested and maintained by NVIDIA, and provides scripts to perform high-performance inference using NVIDIA TensorRT. More information about the TTS system and its training can be found in the
[NVIDIA DeepLearningExamples](https://github.com/NVIDIA/DeepLearningExamples/tree/master/PyTorch/SpeechSynthesis/Tacotron2).

NVIDIA TensorRT is a platform for high-performance deep learning inference. It includes a deep learning inference optimizer and runtime that delivers low latency and high-throughput for deep learning inference applications. After optimizing the compute-intensive acoustic model with NVIDIA TensorRT, inference throughput increased by up to 1.4x over native PyTorch in mixed  precision.

### Software Versions

Software version configuration tested for the instructions that follow:

|Software|Version|
|--------|-------|
|Python|3.6.9|
|CUDA|11.1.1|
|Apex|0.1|
|TensorRT|7.2.3.4|
|PyTorch|1.5.1|


## Quick Start Guide

1. Build and launch the container as described in [TensorRT OSS README](https://github.com/NVIDIA/TensorRT/blob/master/README.md).

    **Note:** After this point, all commands should be run from within the container.

2. Verify TensorRT installation by printing the version:
    ```bash
    python3 -c "import tensorrt as trt; print(trt.__version__)"
    ```

3. Install prerequisite software for TTS sample:
    ```bash
    cd $TRT_OSSPATH/demo/Tacotron2
    sh ./scripts/install_prerequisites.sh
    ```
4. Download pretrained checkpoints from [NGC](https://ngc.nvidia.com/catalog/models) into the `./checkpoints` directory:

- [Tacotron2 checkpoint](https://ngc.nvidia.com/models/nvidia:tacotron2pyt_fp16)
- [WaveGlow checkpoint](https://ngc.nvidia.com/models/nvidia:waveglow256pyt_fp16)

    ```bash
    sh ./scripts/download_checkpoints.sh
    ```

5. Export the models to ONNX intermediate representation (ONNX IR).
   Export Tacotron 2 to three ONNX parts: Encoder, Decoder, and Postnet:

	```bash
	mkdir -p output
	python tensorrt/convert_tacotron22onnx.py --tacotron2 ./checkpoints/nvidia_tacotron2pyt_fp16_20190427 -o output/ --fp16
	```

    Convert WaveGlow to ONNX IR:

	```bash
	python tensorrt/convert_waveglow2onnx.py --waveglow ./checkpoints/nvidia_waveglow256pyt_fp16 --config-file config.json --wn-channels 256 -o output/ --fp16
    ```

	After running the above commands, there should be four new ONNX files in `./output/` directory:
    `encoder.onnx`, `decoder_iter.onnx`, `postnet.onnx`, and `waveglow.onnx`. If TensorRT 8.0+ is being used and the `--no-loop` option is not specified, `decoder.onnx` will also be created.

6. Export the ONNX IRs to TensorRT engines with fp16 mode enabled:

		```bash
	python tensorrt/convert_onnx2trt.py --encoder output/encoder.onnx --decoder output/decoder_iter.onnx --postnet output/postnet.onnx --waveglow output/waveglow.onnx -o output/ --fp16
	```

	After running the command, there should be four new engine files in `./output/` directory:
    `encoder_fp16.engine`, `decoder_iter_fp16.engine`, `postnet_fp16.engine`, and `waveglow_fp16.engine`.

    For TensorRT 8.0+, use `output/decoder.onnx` for the decoder, which will output `decoder_with_outer_loop_fp16.engine`.

7. Run TTS inference pipeline with fp16:

	
	```bash
	python tensorrt/inference_trt.py -i phrases/phrase.txt --encoder output/encoder_fp16.engine --decoder output/decoder_iter_fp16.engine --postnet output/postnet_fp16.engine --waveglow output/waveglow_fp16.engine -o output/ --fp16
	```

    For TensorRT 8.0+, use `decoder_with_outer_loop_fp16.engine` for the decoder. 

## Performance

### Benchmarking

The following section shows how to benchmark the TensorRT inference performance for our Tacotron2 + Waveglow TTS.

#### TensorRT inference benchmark

Before running the benchmark script, please download the checkpoints and build the TensorRT engines for the Tacotron2 and Waveglow models as prescribed in the [Quick Start Guide](#quick-start-guide) above.

The inference benchmark is performed on a single GPU by the `inference_benchmark.sh` script, which runs 3 warm-up iterations then runs timed inference for 1000 iterations.

```bash
bash scripts/inference_benchmark.sh
```

*Note*: For benchmarking we use WaveGlow with 256 residual channels.

### Results

#### Inference performance: NVIDIA T4 (16GB)

|Framework|Batch size|Input length|Precision|Avg latency (s)|Latency std (s)|Latency confidence interval 90% (s)|Latency confidence interval 95% (s)|Latency confidence interval 99% (s)|Throughput (samples/sec)|Speed-up PyT+TRT/TRT|Avg mels generated (81 mels=1 sec of speech)| Avg audio length (s)| Avg RTF|
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
|PyT+TRT|1| 128| FP16| 0.93| 0.15| 1.09| 1.13| 1.49| 169,104| 1.78| 602| 7.35| 7.9|
|PyT    |1| 128| FP16| 1.58| 0.07| 1.65| 1.70| 1.76|  97,991| 1.00| 605| 6.94| 4.4|

#### Inference performance: NVIDIA V100 (16GB)

|Framework|Batch size|Input length|Precision|Avg latency (s)|Latency std (s)|Latency confidence interval 90% (s)|Latency confidence interval 95% (s)|Latency confidence interval 99% (s)|Throughput (samples/sec)|Speed-up PyT+TRT/TRT|Avg mels generated (81 mels=1 sec of speech)| Avg audio length (s)| Avg RTF|
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
|PyT+TRT|1| 128| FP16| 0.63| 0.02| 0.65| 0.66| 0.67| 242,466| 1.78| 599| 7.09| 10.9|
|PyT    |1| 128| FP16| 1.13| 0.03| 1.17| 1.17| 1.21| 136,160| 1.00| 602| 7.10|  6.3|
