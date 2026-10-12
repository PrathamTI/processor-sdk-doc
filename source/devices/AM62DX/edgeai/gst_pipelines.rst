.. _pub_edgeai_am62dx_gst_pipelines:

======================
GStreamer Pipelines
======================

.. toctree::
   :maxdepth: 3

   gst_pipelines/audio_enhancement
   gst_pipelines/audio_classification
   gst_pipelines/pipeline_elements
   gst_pipelines/advanced_usage

Overview
========

The pipelines split work between the ARM core and the C7x DSP:

- **ARM core**: runs GStreamer, reads and writes files, and drives the pipeline.
- **C7x DSP**: runs the STFT, ISTFT and deinterleave/interleave kernels of ``tidspkernel``.
- **TVM runtime**: runs the neural network models of ``titvm``. See
  :ref:`pub_edgeai_am62dx_titvm_execution` for where it runs.
- **RPMsg**: the command channel between the ARM core and the DSP.
- **DMA-BUF**: buffers shared with the DSP, so data is copied only where the pipeline needs it.

Pipelines Shipped with the Plugins
==================================

.. list-table::
   :header-rows: 1
   :widths: 25 35 40

   * - Pipeline
     - Model and input
     - Output
   * - Speech enhancement
     - ``gcrn``, ``input_audio.wav``
     - Enhanced WAV file, optionally played back
   * - Audio event classification
     - ``yamnet``, ``urbansound_16k.wav``
     - Top-k class names and scores printed to the console
   * - Audio classification
     - ``vggish``, ``urbansound_16k.wav``
     - Top-k class names and scores printed to the console

See :ref:`pub_edgeai_am62dx_audio_enhancement` and
:ref:`pub_edgeai_am62dx_audio_classification` for the commands and the stage details.

Data Flow
=========

Speech enhancement uses the full chain of DSP and model stages:

.. code-block:: text

   filesrc → wavparse → caps (S16LE, 16 kHz, mono)
     → tidspkernel stft (0x1020)           DSP: time to spectrum
     → tidspkernel deinterleave (0x1040)   DSP: split real/imaginary
     → titvm gcrn                          model
     → tidspkernel interleave (0x1040)     DSP: recombine real/imaginary
     → tidspkernel istft (0x1030)          DSP: spectrum to time
     → caps (S16LE, 16 kHz, mono) → wavenc → filesink

Classification uses the STFT stage and then the model:

.. code-block:: text

   filesrc → wavparse → caps (S16LE, 16 kHz, mono)
     → tidspkernel stft (0x1020)           DSP
     → titvm yamnet (or vggish)            model
     → fakesink

Each stage's output size must equal the next stage's expected input size. The shape of
the model input is read from ``deploy_graph.json``.

Basic Syntax
============

.. code-block:: bash

   gst-launch-1.0 [-v] element1 property=value ! element2 property=value ! ...

- ``!`` connects the output of the element on its left to the input of the element on its right.
- ``name=`` gives an element a name so later stages can reference it.
- Caps filters such as ``audio/x-raw,format=S16LE,rate=16000,channels=1`` restrict the format
  between two elements.
- Use ``-v`` to print negotiated caps.

The properties of each element are listed in :ref:`pub_edgeai_am62dx_pipeline_elements`.
