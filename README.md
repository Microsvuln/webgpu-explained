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

![Screenshot](https://web-dev.imgix.net/image/vvhSqZboQoZZN9wBvoXq72wzGAf1/q9PYk219Ykt873iQa0Vc.jpeg))



## Shader Programming

Programs running on the GPU that only perform computations (and don't draw triangles) are called compute shaders. 
These shaders are executed in parallel by hundreds of GPU cores (which are smaller than CPU cores) that operate together to crunch data. Their input and outputs are buffers in WebGPU .

To illustrate the use of compute shaders in WebGPU, we'll play with matrix multiplication, a common algorithm in machine learning illustrated below .

![Screenshot]([:/e5c1e41ee59b4abd97720034443686ab](https://web-dev.imgix.net/image/vvhSqZboQoZZN9wBvoXq72wzGAf1/q9PYk219Ykt873iQa0Vc.jpeg?auto=format&w=1428))

In short, here's what we're going to do :

1. Create three GPU buffers (two for the matrices to multiply and one for the result matrix) .
2. Describe input and output for the compute shader .
3. Compile the compute shader code.
4. Setup a compute pipeline .
5. Submit in batch the encoded commands to the GPU.
6. Read the result matrix GPU buffer.

### GPU Buffer creation

For the sake of simplicity, matrices will be represented as a list of floating point numbers. The first element is the number of rows, the second element the number of columns, and the rest is the actual numbers of the matrix.


![c97ac9edca8cf6b6ffaad9fa072b7571.png](:/fed169b41f7d4b65930744236c584fd1)


The three GPU buffers are storage buffers as we need to store and retrieve data in the compute shader. This explains why the GPU buffer usage flags include `GPUBufferUsage.STORAGE` for all of them. The result matrix usage flag also has `GPUBufferUsage.COPY_SRC `because it will be copied to another buffer for reading once all GPU queue commands have all been executed.

```
const adapter = await navigator.gpu.requestAdapter();
if (!adapter) { return; }
const device = await adapter.requestDevice();


// First Matrix

const firstMatrix = new Float32Array([
  2 /* rows */, 4 /* columns */,
  1, 2, 3, 4,
  5, 6, 7, 8
]);

const gpuBufferFirstMatrix = device.createBuffer({
  mappedAtCreation: true,
  size: firstMatrix.byteLength,
  usage: GPUBufferUsage.STORAGE,
});
const arrayBufferFirstMatrix = gpuBufferFirstMatrix.getMappedRange();
new Float32Array(arrayBufferFirstMatrix).set(firstMatrix);
gpuBufferFirstMatrix.unmap();


// Second Matrix

const secondMatrix = new Float32Array([
  4 /* rows */, 2 /* columns */,
  1, 2,
  3, 4,
  5, 6,
  7, 8
]);


const gpuBufferSecondMatrix = device.createBuffer({
  mappedAtCreation: true,
  size: secondMatrix.byteLength,
  usage: GPUBufferUsage.STORAGE,
});
const arrayBufferSecondMatrix = gpuBufferSecondMatrix.getMappedRange();
new Float32Array(arrayBufferSecondMatrix).set(secondMatrix);
gpuBufferSecondMatrix.unmap();


// Result Matrix

const resultMatrixBufferSize = Float32Array.BYTES_PER_ELEMENT * (2 + firstMatrix[0] * secondMatrix[1]);
const resultMatrixBuffer = device.createBuffer({
  size: resultMatrixBufferSize,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC
});
```


### Bind group layout and bind group

Concepts of bind group layout and bind group are specific to WebGPU. A bind group layout defines the input/output interface expected by a shader, while a bind group represents the actual input/output data for a shader.

In the example below, the bind group layout expects two readonly storage buffers at numbered entry bindings 0, 1, and a storage buffer at 2 for the compute shader. The bind group on the other hand, defined for this bind group layout, associates GPU buffers to the entries: `gpuBufferFirstMatrix` to the binding 0, `gpuBufferSecondMatrix` to the binding 1, and resultMatrixBuffer to the binding 2.

```
const bindGroupLayout = device.createBindGroupLayout({
  entries: [
    {
      binding: 0,
      visibility: GPUShaderStage.COMPUTE,
      buffer: {
        type: "read-only-storage"
      }
    },
    {
      binding: 1,
      visibility: GPUShaderStage.COMPUTE,
      buffer: {
        type: "read-only-storage"
      }
    },
    {
      binding: 2,
      visibility: GPUShaderStage.COMPUTE,
      buffer: {
        type: "storage"
      }
    }
  ]
});

const bindGroup = device.createBindGroup({
  layout: bindGroupLayout,
  entries: [
    {
      binding: 0,
      resource: {
        buffer: gpuBufferFirstMatrix
      }
    },
    {
      binding: 1,
      resource: {
        buffer: gpuBufferSecondMatrix
      }
    },
    {
      binding: 2,
      resource: {
        buffer: resultMatrixBuffer
      }
    }
  ]
});
```


### Compute shader code

The compute shader code for multiplying matrices is written in WGSL, the WebGPU Shader Language, that is trivially translatable to SPIR-V. Without going into detail, you should find below the three storage buffers identified with var<storage>. The program will use `firstMatrix` and `secondMatrix` as inputs and resultMatrix as its output.

Note that each storage buffer has a `binding` decoration used that corresponds to the same index defined in bind group layouts and bind groups declared above.

```
const shaderModule = device.createShaderModule({
  code: `
    struct Matrix {
      size : vec2<f32>,
      numbers: array<f32>,
    }

    @group(0) @binding(0) var<storage, read> firstMatrix : Matrix;
    @group(0) @binding(1) var<storage, read> secondMatrix : Matrix;
    @group(0) @binding(2) var<storage, write> resultMatrix : Matrix;

    @stage(compute) @workgroup_size(8, 8)
    fn main(@builtin(global_invocation_id) global_id : vec3<u32>) {
      // Guard against out-of-bounds work group sizes
      if (global_id.x >= u32(firstMatrix.size.x) || global_id.y >= u32(secondMatrix.size.y)) {
        return;
      }

      resultMatrix.size = vec2(firstMatrix.size.x, secondMatrix.size.y);

      let resultCell = vec2(global_id.x, global_id.y);
      var result = 0.0;
      for (var i = 0u; i < u32(firstMatrix.size.y); i = i + 1u) {
        let a = i + resultCell.x * u32(firstMatrix.size.y);
        let b = resultCell.y + i * u32(secondMatrix.size.y);
        result = result + firstMatrix.numbers[a] * secondMatrix.numbers[b];
      }

      let index = resultCell.y + resultCell.x * u32(secondMatrix.size.y);
      resultMatrix.numbers[index] = result;
    }
  `
});
```



