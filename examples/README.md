# DOCA Examples

## DOCA Framework

The DOCA Framework (Data Center-on-a-Chip Architecture) is a software development framework designed by NVIDIA to simplify and accelerate application development on BlueField DPUs. It provides a comprehensive set of APIs, libraries, and runtime services that allow developers to leverage the DPU's hardware accelerators efficiently. By abstracting low-level hardware details, DOCA enables building applications that are optimized for networking, security, storage, and data movement tasks.

## DOCA Sample and Application

DOCA provides a collection of sample applications that showcase how to efficiently utilize BlueField DPUs for various data movement and processing tasks. The DOCA runtime includes several example applications.

In the DOCA framework, Samples and Applications serve different purposes:

* Samples are simplified, focused code examples designed to demonstrate specific features or APIs of DOCA. They are minimal in scope and typically highlight a single concept, making them ideal for learning and quick experimentation.
* Applications are more comprehensive programs that integrate multiple DOCA features to address real-world use cases. They are closer to production-level software and demonstrate how different components can work together to build complete, functional solutions.

![](/assets/doca.png)

## Environment

In this document, we install DOCA 2.9.2.2005. You can check the version with the following commend.

`dpkg -l | grep doca`

![](/assets/doca-bluefield-version.png)

As described above, the DOCA execution environment consists of two distinct components: the DPU and the host. It is important to ensure that the DOCA versions installed on both the DPU and the host are the same. 

Version mismatches between the two environments can lead to compatibility issues, unexpected behavior, or failures when running DOCA applications. Before executing any sample or application, it is recommended to verify the DOCA SDK versions on both sides and update them if necessary to maintain consistency.

## DOCA DMA Sample

Please refer to the [DOCA DMA Sample](./doca-dma-sample.md) for more details.

## Other References

- [DOCA GPU Packet Processing Application Guide](https://docs.nvidia.com/doca/sdk/doca+gpu+packet+processing+application+guide/index.html)
- [DOCA Libraries](https://docs.nvidia.com/doca/sdk/doca+libraries/index.html)
- [DOCA Reference Applications](https://docs.nvidia.com/doca/sdk/doca+reference+applications/index.html)
