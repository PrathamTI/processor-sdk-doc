.. _edgeai-index-am62dx:

***********************************************************
Edge AI
***********************************************************

.. rubric:: Welcome to the Processor SDK Linux Edge AI Software Developer's Guide for AM62Dx

This section describes the GStreamer plugins that run audio processing and neural network
inference on the AM62Dx C7x DSP: ``tidspkernel`` for STFT, ISTFT and deinterleave/interleave,
and ``titvm`` for TVM model inference. It covers installation, the pipelines built on these
plugins, element properties, and troubleshooting.

.. toctree::
   :maxdepth: 3

   getting_started
   gst_pipelines
   faq

|

.. rubric:: What the plugins do

- **tidspkernel** runs STFT, ISTFT and deinterleave/interleave kernels on the C7x DSP.
- **titvm** runs TVM-compiled models, through the model daemon or in-process.
- Both elements exchange data with the DSP over RPMsg. Buffers are shared with DMA
  where the pipeline allows it.

|

.. rubric:: Getting Help

For technical support, please post your questions at `http://e2e.ti.com <http://e2e.ti.com/>`__.
