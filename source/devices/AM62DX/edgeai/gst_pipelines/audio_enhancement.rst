.. _pub_edgeai_am62dx_audio_enhancement:

========================
Audio Enhancement
========================

The speech enhancement pipeline runs the GCRN model on a 16 kHz mono WAV file. It converts
the audio to the frequency domain on the DSP, runs the model, and converts the result back
to audio.

Pipeline Stages
===============

.. code-block:: text

   filesrc → wavparse → audio/x-raw (S16LE, 16 kHz, mono)
     → tidspkernel name=stft         msg-type=0x1020   STFT
     → tidspkernel name=deinterleave msg-type=0x1040   interleave_direction=0
     → titvm                         model-path=.../gcrn
     → tidspkernel name=interleave   msg-type=0x1040   interleave_direction=1
     → tidspkernel name=istft        msg-type=0x1030   ISTFT
     → audio/x-raw (S16LE, 16 kHz, mono) → tee
           → wavenc → filesink       (WAV file)
           → autoaudiosink           (playback)

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - Stage
     - Runs on
     - What it does
   * - ``stft``
     - C7x DSP
     - Converts 16-bit PCM to complex spectral frames. FFT size 320, hop 160 samples,
       401 frames per window, 64 frames per DSP request.
   * - ``deinterleave``
     - C7x DSP
     - Splits the spectral frames into real and imaginary parts for the model.
   * - ``titvm``
     - Model daemon or in-process TVM runtime
     - Runs the GCRN graph. Input and output shape are read from ``deploy_graph.json``
       (``[1, 2, 401, 161]``).
   * - ``interleave``
     - C7x DSP
     - Recombines the real and imaginary parts into the layout ISTFT expects.
   * - ``istft``
     - C7x DSP
     - Converts the enhanced spectrum back to 16-bit PCM.

.. _pub_edgeai_am62dx_enhancement_command:

Command
=======

The command below is the one that runs in ``/opt/edgeai-gst-plugins/pipeline.sh``. It
writes ``/tmp/gst_enhanced.wav`` and plays the same output through ``autoaudiosink``:

.. code-block:: bash

   INPUT=/usr/share/tvm_inference/input/input_audio.wav
   MODEL=/usr/share/tvm_inference/artifacts/gcrn

   gst-launch-1.0 -v \
     filesrc location=$INPUT ! wavparse ! \
     audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tidspkernel name=stft msg-type=0x1020 model-path=$MODEL \
       hop-size=160 fft-size=320 window-frames=401 batch-size=64 \
       max-stream-samples=160480 ! \
     tidspkernel name=deinterleave msg-type=0x1040 interleave_direction=0 \
       model-path=$MODEL fft-size=320 window-frames=401 ! \
     titvm model-path=$MODEL ! \
     tidspkernel name=interleave msg-type=0x1040 interleave_direction=1 \
       model-path=$MODEL fft-size=320 window-frames=401 ! \
     tidspkernel name=istft msg-type=0x1030 model-path=$MODEL \
       hop-size=160 fft-size=320 window-frames=401 batch-size=64 \
       max-stream-samples=160480 ! \
     audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tee name=t \
     t. ! queue ! wavenc ! filesink location=/tmp/gst_enhanced.wav \
     t. ! queue ! audioconvert ! audioresample ! autoaudiosink

For a headless run, remove the ``tee`` branch and keep only the file output:

.. code-block:: bash

   ... ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     wavenc ! filesink location=/tmp/gst_enhanced.wav

.. _pub_edgeai_am62dx_enhancement_chunking:

Why ``max-stream-samples`` Is Required
======================================

``window-frames=401`` is above the 256-frame threshold, so the STFT stage uses
overlap-save chunking. The whole stream is collected in DSP memory before processing, and
``max-stream-samples`` sets how much memory is reserved. If it is missing or too small, the
pipeline fails at start-up or at end of stream.

The minimum value is the padded length of the input:

.. code-block:: text

   chunk_samples = window-frames x hop-size                    = 401 x 160 = 64160
   hop_samples   = (window-frames - overlap-frames) x hop-size = (401 - 100) x 160 = 48160
   n_chunks      = 1 + ceil((N - chunk_samples) / hop_samples)
   padded        = (n_chunks - 1) x hop_samples + chunk_samples

For the 156302-sample ``input_audio.wav`` (N = 156302), ``n_chunks`` is 3 and the padded
length is 160480. Set ``max-stream-samples`` to at least that value.

``overlap-frames`` defaults to 100. The output is trimmed back to the input length.

For a different input file, compute the padded length with the same formula and set
``max-stream-samples`` on both the STFT and ISTFT elements, as the command above does.

Running the Pipeline
====================

1. Check the input and the model:

   .. code-block:: bash

      ls -l /usr/share/tvm_inference/input/input_audio.wav
      ls /usr/share/tvm_inference/artifacts/gcrn/

2. Check that the DSP is running:

   .. code-block:: bash

      cat /sys/class/remoteproc/remoteproc*/state

3. Run the command from :ref:`Command <pub_edgeai_am62dx_enhancement_command>`.

4. Check the output:

   .. code-block:: bash

      ls -lh /tmp/gst_enhanced.wav

The model stage prints the shapes it read from ``deploy_graph.json``:

.. code-block:: text

   [TVM] Auto-detected input shape: [1,2,401,161]
   [TVM] Auto-detected output shape: [1,2,401,161]
   [TVM] Output element count known up front from deploy_graph.json: 129122 floats

Startup Time
------------

``titvm`` first tries the model daemon at ``/var/run/tvm-inference.sock``. If the daemon
does not answer, it waits up to ``daemon-timeout-ms`` (30 seconds by default) before
loading the model in-process. Start the daemon before running the pipeline to avoid that
wait. See :ref:`pub_edgeai_am62dx_titvm_execution`.

Troubleshooting
===============

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Message
     - Action
   * - ``max-stream-samples must be explicitly set (> 0)``
     - Set ``max-stream-samples`` on the STFT element, using the padded length above.
   * - ``padded length ... exceeds dma_input capacity``
     - Increase ``max-stream-samples`` to the padded length.
   * - ``Invalid incoming buffer``
     - The input is longer than ``max-stream-samples``. Increase it.
   * - ``Model daemon handshake failed`` followed by a long wait
     - Start the daemon, or accept the in-process load time.
   * - ``c7x_client_open() failed — is the c7x_compute firmware running?``
     - The C7x firmware is not running. Check ``/sys/class/remoteproc/remoteproc*/state``.
   * - ``DMA heap alloc failed``
     - Not enough contiguous memory for the model. Reboot and retry, then check ``/proc/meminfo``.
   * - ``TI-Offload exchange failed`` or ``DSP kernel failed: remote status=``
     - The DSP firmware does not match the plugin. Check the firmware version.
   * - ``Failed to set pipeline to PAUSED``
     - Run again with ``GST_DEBUG=tidspkernel:6,titvm:6`` and check the first error.
