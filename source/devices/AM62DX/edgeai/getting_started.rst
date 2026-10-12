.. _pub_edgeai_am62dx_getting_started:

===============
Getting Started
===============

This section lists what must be on the board before the GStreamer pipelines run, how to
build and install the plugins, and how to check the installation.

Hardware
========

- AM62Dx EVM with a 16 GB or larger SD card
- 5 V to 20 V power supply
- UART console cable (recommended for logs)
- Audio output (3.5 mm or USB) if you want to play audio with ``autoaudiosink``

For the EVM setup, see the AM62Dx EVM User Guide.

Software Requirements
=====================

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Component
     - Notes
   * - C7x DSP firmware
     - Must be running. Check with ``cat /sys/class/remoteproc/remoteproc*/state``.
       The firmware must provide the TI-Offload generic service that ``tidspkernel`` uses.
   * - ``libti_rpmsg_dma``
     - RPMsg and DMA client library.
   * - TVM runtime
     - ``libtvm_runtime.so`` and a ``tvm.pc`` in the pkgconfig path.
   * - json-c
     - Used by ``titvm`` to parse ``deploy_graph.json``.
   * - GStreamer 1.0
     - ``gstreamer``, ``gst-plugins-base`` (including allocators) and ``gst-inspect-1.0``.
   * - Plugins
     - ``libgstti.so`` in ``/usr/lib/gstreamer-1.0/``. It contains ``tidspkernel``, ``titvm``
       and the RPMsg channel helper.
   * - Model artifacts
     - Under ``/usr/share/tvm_inference/``. See :ref:`pub_edgeai_am62dx_model_artifacts`.

Building on the Board
=====================

The steps below build natively on the board. For a host cross-build, see
:ref:`pub_edgeai_am62dx_cross_build`.

Step 1: Build and install ``libti_rpmsg_dma``
---------------------------------------------

Use the ``offload`` branch. The plugin needs ``ti_rpmsg_rpc_client.h`` and the
``ti_rpmsg_rpc_generic_execute`` symbol from this library:

.. code-block:: bash

   git clone -b offload https://github.com/v-singh1/rpmsg-dma.git
   cd rpmsg-dma/library
   cmake -S . -B build
   cmake --build build
   sudo cmake --install build
   sudo ldconfig

Confirm the install:

.. code-block:: bash

   ls /usr/local/include/ti_rpmsg_rpc_client.h
   nm -D /usr/local/lib64/libti_rpmsg_dma.so | grep ti_rpmsg_rpc_generic_execute

If ``/usr/lib`` already has an older ``libti_rpmsg_dma.so``, the plugin link step picks
that copy up. Replace it with the new build, or the link fails with an undefined reference
to ``ti_rpmsg_rpc_generic_execute``.

Step 2: Build and install the plugins
-------------------------------------

.. code-block:: bash

   git clone -b develop https://github.com/TexasInstruments/edgeai-gst-plugins.git
   cd edgeai-gst-plugins
   export SOC=am62d
   meson setup build --prefix=/usr -Dpkg_config_path=pkgconfig \
     -Dtvm-plugins=enabled -Dstft-plugins=enabled \
     -Ddl-plugins=disabled -Denable-tidl=disabled
   ninja -C build
   sudo ninja -C build install
   sudo ldconfig

Use the branch that contains the TI-Offload change for ``tidspkernel``. That change is not
yet merged into ``develop``, so check the release notes for the version to install.

``SOC`` must be exported in the same shell. ``tvm-plugins`` and ``stft-plugins`` are only
accepted when ``SOC=am62d``.

Step 3: Refresh the GStreamer registry
--------------------------------------

.. code-block:: bash

   rm -rf ~/.cache/gstreamer-1.0/registry.aarch64.bin

.. _pub_edgeai_am62dx_cross_build:

Cross-Building on a Host
------------------------

Build on a host with the SDK toolchain, following the cross-compile steps in the
``edgeai-gst-plugins`` README. Install into the SDK target filesystem with ``DESTDIR``,
then copy the libraries and headers to the board. Run ``ldconfig`` and step 3 on the board.
The board's glibc and GStreamer versions must match the SDK.

Verifying the Installation
==========================

.. code-block:: bash

   gst-inspect-1.0 tidspkernel
   gst-inspect-1.0 titvm

Both commands must print the element details. If an element is missing, check that
``/usr/lib/gstreamer-1.0/libgstti.so`` exists and that ``ldconfig`` has run.

Check the DSP:

.. code-block:: bash

   cat /sys/class/remoteproc/remoteproc*/name
   cat /sys/class/remoteproc/remoteproc*/state
   ls -l /dev/rpmsg*

The C7x remote processor must report ``running``.

.. _pub_edgeai_am62dx_model_artifacts:

Model Artifacts and Labels
==========================

Model directories are under ``/usr/share/tvm_inference/artifacts/``:

- ``gcrn``: speech enhancement
- ``yamnet``: 521-class audio event classification
- ``vggish``: 10-class audio classification

Each model directory must contain:

.. code-block:: text

   deploy_graph.json     graph description; input and output shapes are read from here
   deploy_lib.so         compiled model library
   deploy_param.params   model weights

Label files are under ``/usr/share/tvm_inference/labels/``:

- ``yamnet_label_list.txt``: 521 lines, one class name per line
- ``vggish_label_list.txt``: 10 lines, one class name per line

Sample input files are under ``/usr/share/tvm_inference/input/``.

Input Audio
===========

The pipelines expect 16 kHz, mono, signed 16-bit PCM WAV. Convert other files with ffmpeg:

.. code-block:: bash

   ffmpeg -i input.wav -ar 16000 -ac 1 -sample_fmt s16 output_16k.wav

Or put ``audioconvert ! audioresample`` before the ``audio/x-raw`` capsfilter in the pipeline.

Next Steps
==========

Continue with :ref:`pub_edgeai_am62dx_gst_pipelines` to run the enhancement and
classification pipelines.
