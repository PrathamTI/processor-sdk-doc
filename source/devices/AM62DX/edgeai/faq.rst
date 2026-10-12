.. _pub_edgeai_am62dx_faq:

===
FAQ
===

Setup
=====

**Which elements are provided?**

``tidspkernel`` (STFT, ISTFT, deinterleave/interleave on the C7x DSP) and ``titvm`` (TVM model
inference). Both are in ``/usr/lib/gstreamer-1.0/libgstti.so``.

**How do I check that the plugins are installed?**

.. code-block:: bash

   gst-inspect-1.0 tidspkernel
   gst-inspect-1.0 titvm

If an element is not found, check that ``libgstti.so`` exists in ``/usr/lib/gstreamer-1.0/``,
run ``ldconfig``, and delete the registry cache at ``~/.cache/gstreamer-1.0/``.

**What audio format do the pipelines need?**

16 kHz, mono, signed 16-bit PCM (S16LE). Convert other inputs with ``audioconvert !
audioresample`` before the ``audio/x-raw`` capsfilter, or convert the file with ffmpeg.

**How do I check that the DSP is running?**

.. code-block:: bash

   cat /sys/class/remoteproc/remoteproc*/name
   cat /sys/class/remoteproc/remoteproc*/state

The C7x remote processor must report ``running``.

Running Pipelines
=================

**The enhancement pipeline fails at start-up with "max-stream-samples must be explicitly set".**

The STFT element needs ``max-stream-samples`` when ``window-frames`` is above 256. For
``input_audio.wav`` (156302 samples), use 160480. See
:ref:`pub_edgeai_am62dx_enhancement_chunking` for the formula.

**The pipeline is slow to start and prints "Model daemon handshake failed".**

``titvm`` tries the model daemon first. It waits up to ``daemon-timeout-ms`` (30 seconds by
default) before loading the model in-process. Start the daemon before running the pipeline to
avoid the wait.

**The pipeline prints "c7x_client_open() failed".**

The C7x firmware is not running. Check the remote processor state, as above.

**The pipeline prints "DMA heap alloc failed" while loading a model.**

The contiguous memory for the model is not available. Reboot the board and run the pipeline
again. Check ``grep -i cma /proc/meminfo``.

**The classification pipeline prints nothing.**

Check that ``class-map-path`` is set. Without it, ``titvm`` does not print predictions. Run
with ``GST_DEBUG=titvm:5`` to see the model load.

**Which class names go with YAMNet and VGGish?**

Use ``yamnet_label_list.txt`` (521 lines) with YAMNet and ``vggish_label_list.txt`` (10 lines)
with VGGish. The two lists are not interchangeable.

Models
======

**Which models are provided?**

- ``gcrn``: speech enhancement
- ``yamnet``: 521-class audio event classification
- ``vggish``: 10-class audio classification

**Can I run my own model?**

``titvm`` runs any TVM model directory with the three expected files. The DSP stages depend on
the firmware, and a model with a different spectral layout or DSP processing needs a new
firmware ModelId. See :ref:`pub_edgeai_am62dx_advanced_usage`.

Debugging
=========

**How do I collect logs for a failure?**

.. code-block:: bash

   GST_DEBUG=tidspkernel:6,titvm:6,tirpmsgchan:6 gst-launch-1.0 -v ... 2>&1 | tee debug.log

Include ``debug.log``, the full command, the board's kernel version, and the output of the
remote processor state check in any report.

**Where do I report problems?**

Post questions to the `TI E2E Community Forum <https://e2e.ti.com/>`__.
