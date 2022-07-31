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


### Read buffer memory

Let's see how to copy a GPU buffer to another GPU buffer and read it back .

Since we're writing in the first GPU buffer and we want to copy it to a second GPU buffer, a new usage flag `GPUBufferUsage.COPY_SRC` is required. The second GPU buffer is created in an unmapped state this time with `device.createBuffer()` . its usage flag is `GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ` as it will be used as the destination of the first GPU buffer and read in JavaScript once GPU copy commands have been executed. 

```
// Get a GPU buffer in a mapped state and an arrayBuffer for writing.
const gpuWriteBuffer = device.createBuffer({
  mappedAtCreation: true,
  size: 4,
  usage: GPUBufferUsage.MAP_WRITE | GPUBufferUsage.COPY_SRC
});
const arrayBuffer = gpuWriteBuffer.getMappedRange();

// Write bytes to buffer.
new Uint8Array(arrayBuffer).set([0, 1, 2, 3]);

// Unmap buffer so that it can be used later for copy.
gpuWriteBuffer.unmap();

// Get a GPU buffer for reading in an unmapped state.
const gpuReadBuffer = device.createBuffer({
  size: 4,
  usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ
});
```

Because the GPU is an independent coprocessor, all GPU commands are executed asynchronously. 

 This is why there is a list of GPU commands built up and sent in batches when needed.
 
In WebGPU, the GPU command encoder returned by `device.createCommandEncoder()` is the JavaScript object that builds a batch of "buffered" commands that will be sent to the GPU at some point.
The methods on `GPUBuffer`, on the other hand, are "unbuffered", meaning they execute atomically at the time they are called.

Once you have the GPU command encoder, call `copyEncoder.copyBufferToBuffer()` as shown below to add this command to the command queue for later execution. Finally, finish encoding commands by calling `copyEncoder.finish()` and submit those to the GPU device command queue. The queue is responsible for handling submissions done via `device.queue.submit()` with the GPU commands as arguments. This will atomically execute all the commands stored in the array in order.

`
const copyEncoder = device.createCommandEncoder();
copyEncoder.copyBufferToBufferr(gpuWriteBuffer, 0, gpuReadBuffer, 0, 4);
const copyCommand = copyEncoder.finish();
device.queue.submit([copyCommands]);
`

At this point, GPU queue commands have been sent, but not necessarily executed. To read the second GPU buffer, call `gpuReadBuffer.mapAsync()` with `GPUMapMode.READ`. It returns a promise that will resolve when the GPU buffer is mapped. Then get the mapped range with `gpuReadBuffer.getMappedRange()` that contains the same values as the first GPU buffer once all queued GPU commands have been executed.

```
await gpuReadBuffer.mapAsync(GPUMapMode.READ);
const copyArrayBuffer = gpuReadBuffer.getMappedRange();
console.log(new UintArray(copyArrayBuffer));
```



## Shader Programming

Programs running on the GPU that only perform computations (and don't draw triangles) are called compute shaders. 
These shaders are executed in parallel by hundreds of GPU cores (which are smaller than CPU cores) that operate together to crunch data. Their input and outputs are buffers in WebGPU .

![4aa372343a249bfee6049b572d6eca12.png](:/e5c1e41ee59b4abd97720034443686ab)
