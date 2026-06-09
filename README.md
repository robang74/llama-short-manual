## llama short manual

**`(c)`** 2026 – Roberto A. Foglietta &lt;roberto.foglietta@gmail.com&gt;, CC BY-NC-ND 4.0

- &nbsp;Click on the button to know how to &nbsp;[![Sponsor me](https://img.shields.io/badge/Sponsor-%E2%9D%A4-ff69b4?style=flat&logo=github)](https://github.com/sponsors/robang74)&nbsp; this project and get in touch with me.

A short manual to run AI locally on your PC/laptop w/ decent performance despite minimal HW requisites-

---

### Minimal HW requirements

For a target model between 3B and 4B parameters:

- **CPU**: Intel i5-8265 or Ryzen 2500U
- **RAM**: not less than 16GB w/ Ubuntu
- **GGUFF** file size below the 6GB (cli)

Target quantisation between Q4_K_S and Q5_K_XL

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

To compile your llama native build w/ Vulkan support on Ubuntu:

```sh
sudo apt update
sudo apt install git build-essential cmake mesa-vulkan-drivers \
  vulkan-tools libvulkan-dev vulkan-sdk glslang-tools glslang-dev \
  glslang-tools libopenblas-dev
git clone https://github.com/robang74/llama.cpp
cd llama.cpp
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release -j$(nproc)
```

#### Testing after the build

The `-ngl 0` excludes using the GPU completely, while the `--mlock` keep the model always in RAM:

```
cd build/bin
export PATH=$PWD:$PATH
model="$HOME/Downloads/Qwen3.5-4B-UD-Q4_K_M.gguf"
options="-ctk q8_0 -ctv q8_0 -rea off -fa on --swa-full"
llama-cli --mlock $options -c 4096 -ngl 0 -t 4 -m $model
```

The model is configured to reply without thinking and use in full the flash attention `-fa on --swa-full` which reduces the consumption of the RAM compared the same amount of tokens for the context `-c 4096`.

The context quantisation at 8-bit `-ctk q8_0 -ctv q8_0` keeps a good precision but halves the consumption of the RAM compared with the 16-bit natural representation.

#### Expected performance on i5-8365

Note that off-loading to the GPU is slower than CPU-only:

- `Gemma-2-2b-it.Q4_k_m.gguf` @ 13.3 tk/s
- `Qwen3.5-4B-UD-Q5_K_XL.gguf` @ 5.7 tk/s

This because the i5's GPU cannot handle all the layers.

Threads parallelisation `-t 4` should be related to the number of cores, ignoring the CPU threads. The CPU will throttle a bit above 50%, the performance will be the same, and the CPU will remains relatively colder.
