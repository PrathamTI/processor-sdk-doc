.. _audioai-index:

*********************************************
AM62D Audio AI Software Developer's Guide
*********************************************

.. rubric:: Welcome to the AM62D Audio AI/DL Software Developer's Guide

This guide covers the Audio AI model workflow (model zoo, compile/evaluate/inspect,
performance) as well as the GStreamer plugins that run audio processing and neural
network inference on the AM62Dx C7x DSP: ``tidspkernel`` for STFT, ISTFT and
deinterleave/interleave, and ``titvm`` for TVM model inference.

.. toctree::
   :maxdepth: 5
   :numbered:

   introduction
   model_zoo
   model_compile_evaluate
   model_inspect
   performance
   getting_started
   gst_pipelines
   faq

|

.. rubric:: What the GStreamer plugins do

- **tidspkernel** runs STFT, ISTFT and deinterleave/interleave kernels on the C7x DSP.
- **titvm** runs TVM-compiled models, through the model daemon or in-process.
- Both elements exchange data with the DSP over RPMsg. Buffers are shared with DMA
  where the pipeline allows it.

|

+--------------------------------+--------------------------------------------------------------------------------------------------+
| .. Image:: /images/E2e.jpg     | For technical support please post your questions at `http://e2e.ti.com <http://e2e.ti.com/>`__.  |
+--------------------------------+--------------------------------------------------------------------------------------------------+
