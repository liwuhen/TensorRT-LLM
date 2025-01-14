# **MODEL IS NOT FULLY SUPPORTED YET! DO NOT USE IT.**

# EAGLE speculative Decoding

This document shows how to build and run a model using EAGLE decoding ([`Github`](https://github.com/SafeAILab/EAGLE/tree/main), [`BLOG`](https://sites.google.com/view/eagle-llm)) in TensorRT-LLM on a single node with one GPU or more.

## Overview
Different from other models, EAGLE decoding needs a base model and EAGLE model.

The TensorRT-LLM EAGLE Decoding implementation can be found in [tensorrt_llm/models/eagle/model.py](../../tensorrt_llm/models/eagle/model.py), which actually adds Eagle draft network to a base model.

<!---
For more info about EAGLE visit [speculative decoding documentation](../../docs/source/speculative_decoding.md).
-->

## Known issues
  * Only greedy decoding and greedy acceptance are supported.
  * Only EAGLE-1 is supported.
  * All EAGLE choices have to have exactly the same depth as `num_eagle_layers` of the engine.
  * Cuda graph is not supported yet.
  * Chunked context is not supported yet.
  * Pipeline parallelism is not supported yet.

## Support Matrix
  * GPU Compute Capability >= 8.0 (Ampere or newer)
  * FP16
  * BF16
  * PAGED_KV_CACHE
  * Tensor Parallel

This example focuses on adding EAGLE to LLaMA base model. With some modifications EAGLE can be added to the other base models as well.

## Usage
The TensorRT-LLM EAGLE example code is located in [`examples/eagle`](./). There is one [`convert_checkpoint.py`](./convert_checkpoint.py) file to convert and build the [TensorRT](https://developer.nvidia.com/tensorrt) engine(s) needed to run models with EAGLE decoding support.
In our example, we use the model from HuggingFace [`yuhuili/EAGLE-Vicuna-7B-v1.3`](https://huggingface.co/yuhuili/EAGLE-Vicuna-7B-v1.3), which is a LLAMA based model.

### Build TensorRT engine(s)
Get the weights by downloading the base model [`vicuna-7b-v1.3`](https://huggingface.co/lmsys/vicuna-7b-v1.3) and the EAGLE draft model [`EAGLE-Vicuna-7B-v1.3`](https://huggingface.co/yuhuili/EAGLE-Vicuna-7B-v1.3) from HF.

```
pip install -r requirements.txt

git lfs install
git clone https://huggingface.co/lmsys/vicuna-7b-v1.3
https://huggingface.co/yuhuili/EAGLE-Vicuna-7B-v1.3
```


Here is the example:
```bash
# Convert and Build EAGLE decoding support for vicuna-7b-v1.3
python convert_checkpoint.py --model_dir ./vicuna-7b-v1.3 \
                            --eagle_model_dir EAGLE-Vicuna-7B-v1.3 \
                            --output_dir ./tllm_checkpoint_1gpu_eagle \
                            --dtype float16 \
                            --max_draft_len 63 \
                            --num_eagle_layers 4

# Note: Increasing the batch size may have a negative impact on performance
trtllm-build --checkpoint_dir ./tllm_checkpoint_1gpu_eagle \
             --output_dir ./tmp/eagle/7B/trt_engines/fp16/1-gpu/ \
             --gemm_plugin float16 \
             --use_paged_context_fmha enable \
             --speculative_decoding_mode eagle \
             --max_batch_size 4

# Convert and Build EAGLE decoding support for vicuna-7b-v1.3 with 4-way tensor parallelism.
python convert_checkpoint.py --model_dir ./vicuna-7b-v1.3 \
                            --eagle_model_dir EAGLE-Vicuna-7B-v1.3 \
                            --output_dir ./tllm_checkpoint_4gpu_eagle \
                            --dtype float16 \
                            --max_draft_len 63 \
                            --num_eagle_layers 4 \
                            --tp_size 4 \
                            --workers 4

trtllm-build --checkpoint_dir ./tllm_checkpoint_4gpu_eagle \
             --output_dir ./tmp/eagle/7B/trt_engines/fp16/4-gpu/ \
             --gemm_plugin float16 \
             --use_paged_context_fmha enable \
             --speculative_decoding_mode eagle \
             --max_batch_size 4
```

### Run

```bash
# Eagle decoding using vicuna-7b-v1.3 model with 1 GPU
python ../run.py --engine_dir ./tmp/eagle/7B/trt_engines/fp16/1-gpu/ \
                 --tokenizer_dir ./vicuna-7b-v1.3/ \
                 --max_output_len=100 \
                 --eagle_choices="[[0], [0, 0], [1], [0, 1], [2], [0, 0, 0], [1, 0], [0, 2], [3], [0, 3], [4], [0, 4], [2, 0], [0, 5], [0, 0, 1], [5], [0, 6], [6], [0, 7], [0, 1, 0], [1, 1], [7], [0, 8], [0, 0, 2], [3, 0], [0, 9], [8], [9], [1, 0, 0], [0, 2, 0], [1, 2], [0, 0, 3], [4, 0], [2, 1], [0, 0, 4], [0, 0, 5], [0, 0, 0, 0], [0, 1, 1], [0, 0, 6], [0, 3, 0], [5, 0], [1, 3], [0, 0, 7], [0, 0, 8], [0, 0, 9], [6, 0], [0, 4, 0], [1, 4], [7, 0], [0, 1, 2], [2, 0, 0], [3, 1], [2, 2], [8, 0], [0, 5, 0], [1, 5], [1, 0, 1], [0, 2, 1], [9, 0], [0, 6, 0], [0, 0, 0, 1], [1, 6], [0, 7, 0]]" \
                 --input_text "Once upon"

# Eagle decoding using vicuna-7b-v1.3 model with 4 GPUs
mpirun -np 4 --allow-run-as-root --oversubscribe \
  python ../run.py --engine_dir ./tmp/eagle/7B/trt_engines/fp16/4-gpu/ \
                 --tokenizer_dir ./vicuna-7b-v1.3/ \
                 --max_output_len=100 \
                 --eagle_choices="[[0], [0, 0], [1], [0, 1], [2], [0, 0, 0], [1, 0], [0, 2], [3], [0, 3], [4], [0, 4], [2, 0], [0, 5], [0, 0, 1], [5], [0, 6], [6], [0, 7], [0, 1, 0], [1, 1], [7], [0, 8], [0, 0, 2], [3, 0], [0, 9], [8], [9], [1, 0, 0], [0, 2, 0], [1, 2], [0, 0, 3], [4, 0], [2, 1], [0, 0, 4], [0, 0, 5], [0, 0, 0, 0], [0, 1, 1], [0, 0, 6], [0, 3, 0], [5, 0], [1, 3], [0, 0, 7], [0, 0, 8], [0, 0, 9], [6, 0], [0, 4, 0], [1, 4], [7, 0], [0, 1, 2], [2, 0, 0], [3, 1], [2, 2], [8, 0], [0, 5, 0], [1, 5], [1, 0, 1], [0, 2, 1], [9, 0], [0, 6, 0], [0, 0, 0, 1], [1, 6], [0, 7, 0]]" \
                 --input_text "Once upon"
```

And you will see output like this if run successfully:
```text
......
Input [Text 0]: "<s> Once upon"
Output [Text 0 Beam 0]: "a time, there was a young girl who loved to read. She would spend hours in the library, devouring books of all genres. She had a special love for fairy tales, and would often dream of living in a magical world where she could meet princes and princesses, and have adventures with talking animals.
One day, while she was reading a book, she came across a passage that spoke to her heart. It said, "You are the author of"
```

### Summarization using EAGLE decoding
```bash
# EAGLE decoding using vicuna-7b-v1.3 model with 1 GPU
python ../summarize.py --engine_dir ./tmp/eagle/7B/trt_engines/fp16/1-gpu/ \
                       --hf_model_dir ./vicuna-7b-v1.3/ \
                       --tokenizer_dir ./vicuna-7b-v1.3/ \
                       --test_trt_llm \
                       --data_type fp16 \
                       --eagle_choices="[[0], [0, 0], [1], [0, 1], [2], [0, 0, 0], [1, 0], [0, 2], [3], [0, 3], [4], [0, 4], [2, 0], [0, 5], [0, 0, 1], [5], [0, 6], [6], [0, 7], [0, 1, 0], [1, 1], [7], [0, 8], [0, 0, 2], [3, 0], [0, 9], [8], [9], [1, 0, 0], [0, 2, 0], [1, 2], [0, 0, 3], [4, 0], [2, 1], [0, 0, 4], [0, 0, 5], [0, 0, 0, 0], [0, 1, 1], [0, 0, 6], [0, 3, 0], [5, 0], [1, 3], [0, 0, 7], [0, 0, 8], [0, 0, 9], [6, 0], [0, 4, 0], [1, 4], [7, 0], [0, 1, 2], [2, 0, 0], [3, 1], [2, 2], [8, 0], [0, 5, 0], [1, 5], [1, 0, 1], [0, 2, 1], [9, 0], [0, 6, 0], [0, 0, 0, 1], [1, 6], [0, 7, 0]]" \
                       --batch_size 1

# EAGLE decoding using vicuna-7b-v1.3 with 4 GPUs
mpirun -np 4 --allow-run-as-root --oversubscribe \
    python ../summarize.py --engine_dir ./tmp/eagle/7B/trt_engines/fp16/4-gpu/ \
                           --hf_model_dir ./vicuna-7b-v1.3/ \
                           --tokenizer_dir ./vicuna-7b-v1.3/ \
                           --test_trt_llm \
                           --data_type fp16 \
                           --eagle_choices="[[0], [0, 0], [1], [0, 1], [2], [0, 0, 0], [1, 0], [0, 2], [3], [0, 3], [4], [0, 4], [2, 0], [0, 5], [0, 0, 1], [5], [0, 6], [6], [0, 7], [0, 1, 0], [1, 1], [7], [0, 8], [0, 0, 2], [3, 0], [0, 9], [8], [9], [1, 0, 0], [0, 2, 0], [1, 2], [0, 0, 3], [4, 0], [2, 1], [0, 0, 4], [0, 0, 5], [0, 0, 0, 0], [0, 1, 1], [0, 0, 6], [0, 3, 0], [5, 0], [1, 3], [0, 0, 7], [0, 0, 8], [0, 0, 9], [6, 0], [0, 4, 0], [1, 4], [7, 0], [0, 1, 2], [2, 0, 0], [3, 1], [2, 2], [8, 0], [0, 5, 0], [1, 5], [1, 0, 1], [0, 2, 1], [9, 0], [0, 6, 0], [0, 0, 0, 1], [1, 6], [0, 7, 0]]" \
                           --batch_size 1
```
