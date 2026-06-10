# Llama Short Manual

**`(c)`** 2026 – Roberto A. Foglietta &lt;roberto.foglietta@gmail.com&gt;, CC BY-NC-ND 4.0

- &nbsp;Click on the button to know how to &nbsp;[![Sponsor me](https://img.shields.io/badge/Sponsor-%E2%9D%A4-ff69b4?style=flat&logo=github)](https://github.com/sponsors/robang74)&nbsp; this project and get in touch with me.

A short manual to run AI locally on your PC/laptop w/ decent performance despite minimal HW requisites-

---

### Minimal HW requirements

For a target model between 2B and 8B parameters:

- **CPU**: Intel i5-8265 or Ryzen 2500U
- **RAM**: not less than 16GB w/ Ubuntu
- **DDR4**: clock 2400Mhz in dual-channel
- **GGUF**: file size below 6GB (mem 8GB)

Target quantisation between `Q4_0` (fast) and `Q5_K_XL` (fine).

---

### Memory management plan

The main idea is to split the 16GB RAM in two halves, one for the OS/application usage and the other half for running the AI model.

In this scenario, few point to keep in consideration:

- Using a HTTP server doesn't eat a lot of RAM but the browser.
- The bigger the context size, the smaller the model should be.
- On an Ubuntu essential configuration 8GB can be given to running the AI.

The Q4 and the context are those points on which we can save RAM when the AI local model is supposed to always run in concurrence with a decent desktop activity.

```sh
sudo prlimit --pid=$$ --memlock=$((6<<30)):$((8<<30))
```

For an opimised llama functioning the line above rises the current shell memory allocation limits respectively at 6GB (soft) and 8GB (hard). To make this change permanent for the user without the need to of doing `sudo` each time, the new limits should be set into `/etc/security/limits.conf`.

---

### Native llama quick build

Compile your own llama binary set isn't mandatory for the most common OS configurations:

- [llama.cpp releases by ggml.org](https://github.com/ggml-org/llama.cpp/releases) for Ubuntu, MacOS and Windows,
- supporting: CPU-only, Vulkan, ROCm, OpenVINO, CUDA (*) and HIP
- (*) CUDA support available only for Windows x64 otherwise llamafile

To compile your llama native build w/ Vulkan support on Ubuntu (22.04 LTS, in my case):

```sh
sudo apt update
sudo apt install git build-essential cmake libvulkan1 \
  mesa-vulkan-drivers vulkan-tools libvulkan-dev vulkan-sdk \
  glslang-tools glslang-dev glslang-tools libopenblas-dev \
  pciutils libcurl4-openssl-dev
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
cmake --build build --config Release -j --clean-first
```

The OpenBLAS library is installed because supported but disable because it may cause speed regression compare the llama's ggml-cpu native backend.

---

### Testing after the build

The `-ngl 0` excludes using the GPU completely, while the `--mlock` keep the model always in RAM:

```sh
cd build/bin
# Use this to call these binaries from anywhere
export PATH=$PWD:$PATH

model="$HOME/Downloads/Qwen3.5-4B-Q4_K_M.gguf"
opts="-ctk q8_0 -ctv q8_0 -ngl 0 --mlock --mmap -t 4"
./llama-cli $opts -c 4096 -rea off -fa on -m $model
```

The model is configured to reply without thinking and use in full the flash attention `-fa on` which reduces the consumption of the RAM compared the same amount of tokens for the context `-c 4096`.

The context quantisation at 8-bit `-ctk q8_0 -ctv q8_0` keeps a good precision but halves the consumption of the RAM compared with the 16-bit natural representation.

Moreover, for telegraphic-style answering mode, append:

- `-sys "You are a helpful assistant. Be concise in answering."`

However this system prompt strongly influences the tests in a way that are much less comparable among different users / seeds, therefore it has not been used.

---

### Running the llama server

Starting with the same for running llama-cli enviroment:

```sh
./llama-server $opts -c 4096 -rea off -fa on -m $model \
  --offline --temperature 0.5 -np 1 --cache-ram 0
```

The last two options save RAM because the webserver is limited in running a single AI instance instead of the common four (parallelism), which each of them requires a context window cache allocated.

The server can be started manually or as system service or at the user login time, and the AI chatbot can be accessed at `http://127.0.0.1:8080` by a web browser.

```sh
sudo apt install surf
aisurf() { GDK_BACKEND=x11 surf http://127.0.0.1:8080; }
aisurf
```

Including the minimalistic surf that has a very low RAM footprint (128MB for the whole browser, but a Chrome tab would be similar if already opened for other indispensable activities) and it is safe to use for browsing a fully trusted local address like the one provided by the llama web server.

---

### Expected performance on i5-8365

Testing prompt:

> What is the name of the capital of France?

Note that off-loading to the GPU is slower than CPU-only because the i5's GPU cannot handle all the layers:

| # | Model Name  | Size | Read | Write | Peak | Mem | File | Fit |
|:-:| ----------- |:---:|:----:|:-----:|:----:|:---:|:----:|:---:|
| | | eq. | tk/s | tk/s | GB | GB | GB | |
| 1 | `Qwen-3.5 4B Q4_K_M` [gguf](https://huggingface.co/unsloth/Qwen3.5-4B-MTP-GGUF/resolve/main/Qwen3.5-4B-Q4_K_M.gguf) | 4B | $${\color{lightgreen}\textbf{》26.8《}}$$ | 8.1 | 5.03 | 3.64 | 2.64 | 🟢 |
| 2 | `Gemma-4 E4B-it-QAT Q4_0` [gguf](https://huggingface.co/google/gemma-4-E4B-it-qat-q4_0-gguf/resolve/main/gemma-4-E4B_q4_0-it.gguf) | (8B) | 22.5 | 9.0 | $${\color{lightgray}\textbf{》7.53《}}$$ | 7.14 | $${\color{lightgray}\textbf{》4.80《}}$$ | ✔️ |
| 2 | `Gemma-4 E4B-it-QAT Q4_0` [gguf](https://huggingface.co/google/gemma-4-E4B-it-qat-q4_0-gguf/resolve/main/gemma-4-E4B_q4_0-it.gguf) &nbsp;($${\color{lightgreen}\textbf{full 32K @Q4{\\_}0}}$$) | (8B) | $${\color{lightgreen}\textbf{》26.4《}}$$ | $${\color{lightgreen}\textbf{》9.2《}}$$ | $${\color{lightgreen}\textbf{》7.99《}}$$ | 7.64 | 4.80 | ✅ |
| 3 | `Gemma-4 E4B-it-qat UD-Q4_K_XL` [gguf](https://huggingface.co/unsloth/gemma-4-E4B-it-qat-GGUF/resolve/main/gemma-4-E4B-it-qat-UD-Q4_K_XL.gguf) | (8B) | 20.5 | $${\color{lightgreen}\textbf{》9.2《}}$$ | 6.99 | 6.53 | 3.93 | 🟢 |
| 4 | `Gemma-4 E4B-it-obliterated Q4_K_M` | (8B) | 23.2 | 7.5 | 7.40 | 6.78 | 4.97 | — |
| 5 | `Qwen-3.5 4B UD-Q5_K_XL` [gguf](https://huggingface.co/unsloth/Qwen3.5-4B-MTP-GGUF/resolve/main/Qwen3.5-4B-UD-Q5_K_XL.gguf) | 4B | $${\color{lightgray}\textbf{》18.3《}}$$ | 7.3 | $${\color{lightgreen}\textbf{》4.15《}}$$ | 3.65 | 3.08 | ✔️ |
| 6 | `DeepSeek-R1-distill-Qwen 7B-uncensored i1-Q4_0` | 7B |  15.9 | 6.6 | **8.01** | 7.60 | 4.14 | — |
| 7 | `Apertus 8B-instruct-2509 UD-Q4_K_XL` [gguf](https://huggingface.co/unsloth/Apertus-8B-Instruct-2509-GGUF/resolve/main/Apertus-8B-Instruct-2509-UD-Q4_K_XL.gguf) | 8B | 13.7 | $${\color{lightblue}\textbf{》5.3《}}$$ | 7.61 | 7.27 | 4.78 | ☑️ |
| | | | | | | |
| | *By Comparison*: | | | | | |
| A | `Gemma-2 2B-it Q4_K_M` | 2B | 47.4 | 13.8 | 3.13 | 2.15 | 1.59 | — |
| B | `Qwen-3.5 4B Q5_K_S` [gguf](https://huggingface.co/unsloth/Qwen3.5-4B-GGUF/resolve/main/Qwen3.5-4B-Q5_K_S.gguf) ➡ [llamafile](https://docs.mozilla.ai/llamafile/getting-started/pre-built-llamafiles) &nbsp;($${\color{lightgray}\textbf{size 3.75 GB}}$$) | 4B | 18.5 | 5.0 | **8.80** | 4.78 | 3.02 | ✔️ |
| C | `Qwen-3.5 4B-MTP Q5_K_S` [gguf](https://huggingface.co/unsloth/Qwen3.5-4B-MTP-GGUF/resolve/main/Qwen3.5-4B-Q5_K_S.gguf) | 4B | 16.0 | 7.1 | 3.91 | 3.39 | 2.91 | ✔️ |
| | | | | | | |
| | *Above Limits*: | | | | | |
| 8 | `Gemma-4 12B-it UD-Q4_K_XL` [gguf](https://huggingface.co/unsloth/gemma-4-12b-it-GGUF/resolve/main/gemma-4-12b-it-UD-Q4_K_XL.gguf) &nbsp;($${\color{orange}\textbf{mem. 12 GB}}$$) | 12B | 8.9 | $${\color{orange}\textbf{》3.6《}}$$ | 12.2 | 11.6 | 6.86 | 🔶 |
| 9 | `Gemma-4 12B-it-QAT Q4_0` [gguf](https://huggingface.co/google/gemma-4-12B-it-qat-q4_0-gguf/resolve/main/gemma-4-12b-it-qat-q4_0.gguf) &nbsp;($${\color{orange}\textbf{mem. 14 GB}}$$) | 12B | 9.1 | $${\color{orange}\textbf{》4.0《}}$$ | 13.1 | 11.9 | 6.50 | 🔶 |

#### Table's Notes

- The human reading speed in English varies between 5 and 11 tk/s, on average 7.5 tk/s.
- Some models are more verbose and their Wt/k drop, hence verbosity is a fair penality.
- Energy saving mode (max 15W TDP) otherwise i5-8365 gets hot and drops the frequency.
- Tests were completed before adding `--mmap`, which by defaut is enabled.

#### Data Evaluation

The prompt reading is usually faster (Rtk/s) than generation (Wtk/s) while the RAM consumption, analyzed via free, reveals the full impact of the model file and the context overhead (around 500-600MB extra). This wasn't obvious but `free` output remains consistent across various runs.

Threads parallelisation `-t 4` should be related to the number of cores, ignoring the CPU threads. The CPU will throttle a bit above 50%, the performance will be the same, and the CPU will remains relatively colder and not fully busy.

While the `Q4_0` might seems obsolete, it is way faster when the model is relatively big (7B) and the CPU is relatively old (i5-8th). In some models, distillation (or pruning) and uncensoring (or ablation) can spare a lot of RAM and improve speed.

#### By Comparison

I did as equivalent as possible tests on `Qwen3.5-4B-Q5_K_S.llamafile` and the most significative differences are: 1) it seems faster in loading the model in `--chat` mode; 2) much more pressure on the system RAM, not because the model rather than binary code redundancy; 3) apparently slower in answering. BTW, statistics are required to support these three claims.

Gemma 4's memory values collected are aligned with Google [specifications](https://ai.google.dev/gemma/docs/core#gemma-4-inference-memory-requirements). Hence, the `E4B` is equivalent to a **`8B`** w/o the computational burden of a larger model. The most relevant aspect is about Gemma-4 [Quantization Aware Training](https://huggingface.co/google/gemma-4-E4B-it-qat-q4_0-gguf) which suggests using `Q4_0` for the KV caches is **natively fine**.

Therefor `-ctk q4_0 -ctv q4_0` allowing a relatively huge 32K context window `-c $((32<<10)) --swa-full` while keeping the RAM usage within the 8GB limit. Instead, with the `Gemma-4 12B-it-QAT Q4_0` and `-ctk q4_0 -ctv q4_0` the most daring config is `-c 4096 --swa-full` within the 14GB limit.

#### Conclusions

Considering a machine equipped with a Ryzen Pro 5 series 5000 and 16GB DDR4 3200Mhz in dual-channel, we can easily reach the conclusion that in the range `[ €180, €260 ]` such machine can work as a dedicated uAI-server providing a 12B model access by network, cabled or wifi indifferently.

For running a basic Linux server 2GB of RAM is an "abundant luxury", therefore the Llama memory limits can be raised to 12GB (soft) and 14GB (hard). Considering the overall ratio in computational capacity, twice a Thinkpad X390, the expected throughput is 7.2 tk/s, in CPU-only mode and without specific low-level or ML optimisations.

Finally, looking at the llama pre-built [releases](#native-llama-quick-build), we can note that llamafile exists primarily for supporting CUDA on whatever OS apart from Windows x64 is a real pain. While llamafile with its 0.75GB extra on top of each AI model, detects and compiles on the fly the essential stuff for providing llama the support to deal with a local CUDA installation, if any.

---

### Model loading time

A quick way to test the start time which includes the model loading is to pass as the first prompt the exit command. In this way it is possible to compare the starting time among various models and by a fair comparison with llamafile using the same model:

```sh
drpc() { sync; echo 3 | sudo tee /proc/sys/vm/drop_caches; }

topt="$opts -c 4096 -rea off -fa on"

drpc; time -p ./llama-cli $topt -p "/exit" \
  -m Qwen3.5-4B-Q4_K_M.gguf

drpc; echo "/exit" | time -p sh \
  ./Qwen3.5-4B-Q5_K_S.llamafile --chat $topt
```

As anticipated the llamafile is faster at start-up time:

| Model    | Size   | Quant.  | File | Real | User | Sys  | Range |
| -------- |:------ |:------- |:----:|:----:|:----:|:----:|:-----:|
| Qwen-3.5 | 4B     | Q4_K_M  | 2.64 | 5.64 | 3.94 | 1.51 | min   |
|          |        |         |      | 5.78 | 4.08 | 1.75 | max   |
|          |        |         |      |      |      |      |       |
| Qwen-3.5 | 4B-UD  | Q5_K_XL | 3.08 | 4.64 | 3.01 | 1.40 | min   |
|          |        |         |      | 4.85 | 3.12 | 1.62 | max   |
|          |        |         |      |      |      |      |       |
| Qwen-3.5 | 4B-MTP | Q5_K_S  | 2.91 | 4.09 | 2.91 | 1.18 | min   |
|          |        |         |      | 4.20 | 3.00 | 1.27 | max   |
|          |        |         |      |      |      |      |       |
| **.llamafile**:   |  |  | **3.75** |      |      |      |       |
| Qwen-3.5 | 4B     | Q5_K_S  | 3.02 | 3.88 | 0.76 | 1.59 | min   |
|          |        |         |      | 4.00 | 1.08 | 1.81 | max   |

The user timings aren't comparable with the .llama one due to `sh` use.

---

### Benchmark screenshot example

The correct full approach includes checking also the resident size in memory of the `llama` instance running the model:

```sh
pmem() { grep ^Vm /proc/$(pgrep $1)/status; }
pmem llama-cli | grep VmPeak
```

But dropping the cache before the run, and checking the `free` difference is the most straightforward way to check the `pmem` output:

```sh
$ drpc; free; ./llama-cli --mlock $options -c 4096 -ngl 0 -m $model;
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

