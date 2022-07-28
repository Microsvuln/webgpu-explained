# webgpu-explained
Detailed explanation of WebGPU


# Fuzzing webgpu

Useful links to start with :

`https://alain.xyz/blog/raw-webgpu`

` https://www.willusher.io/graphics/2021/08/29/0-to-gltf-triangle `



### Webgpu Specifications

Initializing webgpu and checking if it will be supported :

```
const entry: GPU = navigator.gpu;
if (!entry) {
    throw new Error('WebGPU is not supported on this browser.');
}
```

### Access GPU using javascript

To access the GPU calling `navigator.gpu.requestAdapter()` is enough.Once you have the GPU adapter, call `adapter.requestDevice()` to get a promise that will resolve with a GPU device you'll use to do some GPU computation.

```
const adapter = await navigator.gpu.requestAdapter();
if (!adapter) { return; }
const device = await adapter.requestDevice();
```

