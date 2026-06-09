# llama short manual

**`(c)`** 2026 – Roberto A. Foglietta &lt;roberto.foglietta@gmail.com&gt;, CC BY-NC-ND 4.0

- &nbsp;Click on the button to know how to &nbsp;[![Sponsor me](https://img.shields.io/badge/Sponsor-%E2%9D%A4-ff69b4?style=flat&logo=github)](https://github.com/sponsors/robang74)&nbsp; this project and get in touch with me.

A short manual to run AI locally on your PC/laptop w/ decent performance despite minimal HW requisites-

---

### Minimal HW requirements

For a target model between 2B and 7B parameters:

- **CPU**: Intel i5-8265 or Ryzen 2500U
- **RAM**: not less than 16GB w/ Ubuntu
- **DDR4**: clock 2400Mhz in dual-channel
- **GGUF** file size below the 6GB (cli)

Target quantisation between `Q4_0` and `Q5_K_XL`

---

### Memory management plan

The main idea is to split the 16GB RAM in two halves, one for the OS/application usage and the other half for running the AI model.

In this scenario, few point to keep in consideration:

- Using a HTTP server doesn't eat a lot of RAM but the browser.
- The bigger the context size, the smaller the model should be.
- On an Ubuntu essential configuration 8GB can be given to running the AI.

The Q4 and the context are those points on which we can save RAM when the AI local model is supposed to always run in concurrence with a decent desktop activity.

---

### Native llama quick build

To compile your llama native build w/ Vulkan support on Ubuntu (22.04 LTS, in my case):

```sh
sudo apt update
sudo apt install git build-essential cmake libvulkan1 \
  mesa-vulkan-drivers vulkan-tools libvulkan-dev vulkan-sdk \
  glslang-tools glslang-dev glslang-tools libopenblas-dev
# optional and useless with -DLLAMA_SERVER_LLAMAUI=OFF
# sudo apt install nodejs npm

# copy 1:1 of the original project, currently at #e3471b3e7
git clone https://github.com/robang74/llama.cpp
cd llama.cpp

# if your GPU does not perform better than '-ngl 0' then
# set Vulkan OFF, add -DGGML_CPU_K_QUANTS=ON and before
# rebuild do rm -rf build to clean the previous build
# which is not strictly necessary but it 100% works.
cmake -B build -DGGML_VULKAN=ON -DGGML_BLAS=OFF \
  -DGGML_NATIVE=ON -DGGML_CPU_K_QUANTS=ON
cmake --build build --config Release -j$(nproc)
```

The OpenBLAS library is installed because supported but disable because it may cause speed regression compare the llama's ggml-cpu native backend.

---

### Testing after the build

The `-ngl 0` excludes using the GPU completely, while the `--mlock` keep the model always in RAM:

```sh
cd build/bin
export PATH=$PWD:$PATH
model="$HOME/Downloads/Qwen3.5-4B-Q4_K_M.gguf"
options="-ctk q8_0 -ctv q8_0 -rea off -fa on -t 4"
llama-cli --mlock $options -c 4096 -ngl 0 -m $model
```

The model is configured to reply without thinking and use in full the flash attention `-fa on` which reduces the consumption of the RAM compared the same amount of tokens for the context `-c 4096`.

The context quantisation at 8-bit `-ctk q8_0 -ctv q8_0` keeps a good precision but halves the consumption of the RAM compared with the 16-bit natural representation.

---

### Expected performance on i5-8365

Testing question:

> What is the name of the capital of France?

Note that off-loading to the GPU is slower than CPU-only because the i5's GPU cannot handle all the layers:

- `Gemma-2-2b-it.Q4_k_m.gguf`: 47.4 Rt/s, 13.8 Wt/s, 2.15/1.59 GB
- [`Qwen3.5-4B-UD-Q5_K_XL.gguf`](https://huggingface.co/unsloth/Qwen3.5-4B-MTP-GGUF/resolve/main/Qwen3.5-4B-UD-Q5_K_XL.gguf?download=true): 18.3 Rtk/s, 6.7 tk/s, 3.65/3.08 GB
- [`Qwen3.5-4B-Q4_K_M.gguf`](https://huggingface.co/unsloth/Qwen3.5-4B-MTP-GGUF/resolve/main/Qwen3.5-4B-Q4_K_M.gguf?download=true): 26.8 Rt/s, 8.1 Wt/s, 3.64/2.64 GB

The prompt reading is usually faster (Rtk/s) than generation (Wtk/s) while the RAM consumption (`available` difference), taken after a Q/A, is a fraction of the model file size. This wasn't obvious but `free` output remains consistent across various runs.

Threads parallelisation `-t 4` should be related to the number of cores, ignoring the CPU threads. The CPU will throttle a bit above 50%, the performance will be the same, and the CPU will remains relatively colder and not fully busy.

- `DeepSeek-R1-Distill-Qwen-7B-Uncensored.i1-Q4_0.gguf`:  15.9 Rt/s, 6.6 Wtk/s, 4.55/4.14 GB, 

While the `Q4_0` might seems obsolete, it is way faster when the model is relatively big (7B) and the CPU is relatively old (i5-8th). In some models, distillation (or pruning) and uncensoring (or ablation) can spare a lot of RAM and improve speed.

---

### Benchmark screenshot example

The correct full approach includes checking also the resident size in memory of the `llama` running the model (checking the `free` difference, also) and dropping the cache before the run:

```sh
$ sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"
$ free; llama-cli --mlock $options -c 4096 -ngl 0 -m $model;
               total        used        free      shared  buff/cache   available
Mem:        16148684     5863108     8595136     1146284     1690440     8827828
Swap:              0           0           0
```
```
Loading model...  


▄▄ ▄▄
██ ██
██ ██  ▀▀█▄ ███▄███▄  ▀▀█▄    ▄████ ████▄ ████▄
██ ██ ▄█▀██ ██ ██ ██ ▄█▀██    ██    ██ ██ ██ ██
██ ██ ▀█▄██ ██ ██ ██ ▀█▄██ ██ ▀████ ████▀ ████▀
                                    ██    ██
                                    ▀▀    ▀▀

build      : b9571-e3471b3e7
model      : Qwen3.5-4B-Q4_K_M.gguf
modalities : text

available commands:
  /exit or Ctrl+C     stop or exit
  /regen              regenerate the last response
  /clear              clear the chat history
  /read <file>        add a text file
  /glob <pattern>     add text files using globbing pattern


> What is the name of the capital of France?

The capital of France is **Paris**.

[ Prompt: 19.4 t/s | Generation: 7.5 t/s ]
```
```sh
(another console)$ free
               total        used        free      shared  buff/cache   available
Mem:        16148684     6166552     5072540     1266792     4909592     6403060
```
```
> /exit
```
