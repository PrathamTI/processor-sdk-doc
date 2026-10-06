.. _pub_edgeai_am62dx_pipeline_elements:

==================
Pipeline Elements
==================

The two AM62Dx plugin elements, ``tidspkernel`` and ``titvm``, are built into one library,
``/usr/lib/gstreamer-1.0/libgstti.so``. The standard GStreamer elements used with them are
listed at the end.

tidspkernel
===========

``tidspkernel`` runs STFT, ISTFT and deinterleave/interleave on the C7x DSP. The operation is
selected by ``msg-type``.

Operations
----------

.. list-table::
   :header-rows: 1
   :widths: 15 25 30 30

   * - ``msg-type``
     - Operation
     - Input
     - Output
   * - ``0x1020``
     - STFT
     - S16LE mono PCM, ``hop-size`` samples per frame
     - ``model-elems`` floats per frame
   * - ``0x1030``
     - ISTFT
     - ``model-elems`` floats per frame
     - S16LE mono PCM, ``hop-size`` samples per frame
   * - ``0x1040``
     - Deinterleave or interleave
     - ``2 x window-frames x (fft-size/2+1)`` floats
     - Same size, reordered. ``interleave-direction`` selects 0 (deinterleave) or 1 (interleave).

If ``msg-type`` is not set, the element picks the operation from its name. The name must end
with ``stft``, ``istft``, ``deinterleave`` or ``interleave``.

Properties
----------

.. list-table::
   :header-rows: 1
   :widths: 22 12 14 52

   * - Property
     - Type
     - Default
     - Meaning
   * - ``msg-type``
     - uint
     - 0 (from name)
     - Operation. See the table above.
   * - ``fft-size``
     - uint
     - 0
     - FFT length in samples. Sets the bin count for deinterleave/interleave, and sets
       ``model-elems`` when no known model matches. The STFT and ISTFT requests do not send it.
   * - ``hop-size``
     - uint
     - 0
     - Samples per frame. Needed for STFT and ISTFT.
   * - ``window-frames``
     - uint
     - 0
     - Frames per window. Values above 256 turn on overlap-save chunking.
   * - ``batch-size``
     - uint
     - 0
     - Frames per DSP request. 0 uses ``window-frames``.
   * - ``chunking-mode``
     - int
     - -1
     - -1 chunks when ``window-frames`` is above 256. 0 never chunks. 1 always chunks.
   * - ``overlap-frames``
     - uint
     - 100
     - Overlap between chunks. Must be less than ``window-frames``.
   * - ``max-stream-samples``
     - uint
     - 0
     - Largest input stream, in samples, for overlap-save chunking. Required when chunking
       is on. See :ref:`pub_edgeai_am62dx_enhancement_chunking`.
   * - ``model-path``
     - string
     - empty
     - Model directory. Its path is matched against the known-model table below.
   * - ``selected-model``
     - uint
     - 2
     - Firmware ModelId sent with each STFT and ISTFT request. Overwritten by the known-model
       table when ``model-path`` matches.
   * - ``model-elems``
     - uint
     - 0
     - Floats per spectral frame. 0 uses the known-model table, or ``(fft-size/2+1) x 2``.
   * - ``interleave-direction``
     - uint
     - 0
     - For ``msg-type=0x1040``: 0 deinterleave, 1 interleave.

The robot test and the enhancement pipeline use ``interleave_direction`` (underscore). Use the
same spelling as the working command you copy.

Known-Model Table
-----------------

``model-path`` is matched, ignoring case, against the directory path. A match sets
``selected-model`` and ``model-elems``.

.. list-table::
   :header-rows: 1
   :widths: 25 25 25

   * - Name in path
     - ``selected-model``
     - ``model-elems``
   * - ``dccrn``
     - 0
     - 514
   * - ``gtcrn``
     - 1
     - 514
   * - ``gcrn``
     - 2
     - 322
   * - ``vggish``
     - 3
     - 64
   * - ``yamnet``
     - 4
     - 64

A path that matches no name keeps the property values. The element logs nothing in that case,
so set ``selected-model`` and ``model-elems`` explicitly for a custom model.

DSP Requests
------------

``tidspkernel`` sends each request using the TI-Offload generic service, on endpoint 13 of
the C7x DSP (processor ID 8). Each request carries:

- A kernel ID: 1 for STFT, 2 for ISTFT, 3 for deinterleave/interleave.
- One input and one output buffer, each given by a 64-bit physical address and a size.
- Params:

  - STFT and ISTFT: ``selected_model``, ``input_frame``, ``output_frame``. The two frame counts
    are equal.
  - Deinterleave/interleave: ``input_frame``, ``fft_size``, ``flag``.

The reply carries only a header. The plugin treats the frame count it requested as the frame
count it got back.

titvm
=====

``titvm`` runs a TVM-compiled model. Input and output shapes are read from the model's
``deploy_graph.json``. Each buffer is checked against the input size, and a mismatch is an
error.

Properties
----------

.. list-table::
   :header-rows: 1
   :widths: 22 12 14 52

   * - Property
     - Type
     - Default
     - Meaning
   * - ``model-path``
     - string
     - empty
     - Directory with ``deploy_graph.json``, ``deploy_lib.so`` and ``deploy_param.params``.
   * - ``class-map-path``
     - string
     - empty
     - Label file. Setting it turns on prediction printing. Empty turns it off.
   * - ``top-k``
     - uint
     - 3
     - Predictions printed per window when ``class-map-path`` is set. Range 1 to 521.
   * - ``daemon-timeout-ms``
     - uint
     - 30000
     - Time allowed for each request and reply with the model daemon, in milliseconds.

Model Directory
---------------

.. code-block:: text

   /usr/share/tvm_inference/artifacts/yamnet/
   ├── deploy_graph.json     graph; input and output shapes are read from attrs.shape
   ├── deploy_lib.so         compiled model library
   └── deploy_param.params   model weights

.. _pub_edgeai_am62dx_titvm_execution:

Execution Path
--------------

When the element starts, it first connects to the model daemon at
``/var/run/tvm-inference.sock``. It sends a PING, waits for the reply, then sends the model
path to be loaded.

- **Daemon available:** every inference goes to the daemon.
- **Daemon not available:** the element waits up to ``daemon-timeout-ms``, then loads the model
  in-process. The board log shows the in-process load opens the C7x client
  (``c7x_client_open()``), so the in-process path also needs the C7x firmware running.
- **Daemon fails during inference:** the element returns an error and does not fall back.

Standard GStreamer Elements Used
================================

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Element
     - Use in these pipelines
   * - ``filesrc``
     - Reads the input file. ``location=`` sets the path.
   * - ``wavparse``
     - Parses a WAV file into PCM.
   * - ``audioconvert`` / ``audioresample``
     - Converts other formats and sample rates to the format the pipeline expects.
   * - ``audio/x-raw`` capsfilter
     - Pins the format, for example ``audio/x-raw,format=S16LE,rate=16000,channels=1``.
   * - ``tee`` and ``queue``
     - Split the stream into branches. Each branch needs its own ``queue``.
   * - ``wavenc`` and ``filesink``
     - Writes a WAV file. ``filesink location=`` sets the path.
   * - ``autoaudiosink``
     - Plays audio through the default output.
   * - ``fakesink``
     - Discards the data. Used for classification, where the output is the console log.

Debugging
=========

.. code-block:: bash

   GST_DEBUG=tidspkernel:6,titvm:6,tirpmsgchan:6 gst-launch-1.0 ...

``tidspkernel`` logs its message type, buffer sizes and each DSP request. ``titvm`` logs the
daemon handshake, the shape it detected, and the model load. ``tirpmsgchan`` logs the RPMsg
channel. For a pipeline graph, set ``GST_DEBUG_DUMP_DOT_DIR``.
