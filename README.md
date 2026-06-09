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

cd build/bin
export PATH=$PWD:$PATH
model=$HOME/Downloads/Qwen3.5-4B-UD-Q5_K_XL.gguf
llama-cli --mlock -ctk q8_0 -ctv q8_0 -rea off -fa on -c 4096 -m $model


