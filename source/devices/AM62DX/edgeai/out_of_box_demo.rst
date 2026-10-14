.. _am62dx_edgeai_oob_demo:

==========================
Out-of-Box Web GUI Demo
==========================

The AM62DX EVM ships with an out-of-box web-based GUI that demonstrates audio AI
capabilities directly from a browser. Once the board is powered on and connected to
a network, the web portal can be accessed from any device on the same network without
any additional setup.

The portal showcases the three core audio AI pipelines — home screen, speech
enhancement, and audio classification — running on the C7x DSP via the TVM runtime.

Home Screen
===========

.. figure:: /images/am62dx_web_gui_home.png
   :align: center

   AM62DX out-of-box web GUI home screen

This is the Sitara AM62D Edge AI Portal, a web-based user interface for managing and
evaluating Texas Instruments' AM62D Evaluation Module. The dashboard allows developers
to launch real-time audio analytics and DSP compute demos, inspect AI models via the
TIDL, and monitor live system telemetry like CPU load, memory footprints, and SDK software
versions.

To open the EdgeAI portal, follow below steps:

1. Once the Am62D edgeai image is flashed (See :ref:`pub_edgeai_am62dxx_flashing`), get the ip address of the evm using:  ip addr
2. Open browser and open link:   <ip-addr>:3000
3. This will start the webserver demo.

The left navigation sidebar groups the portal into three sections:

- **Demos** — ``Audio Intelligence`` (speech enhancement and audio classification) and
  ``DSP Acceleration`` (2D FFT, biquad filter chain, and workload offload via RPMsg-DMA
  compute demos).
- **Tools** — ``AI Model Inspector`` to upload and browse TIDL model inspection reports,
  including architecture, layer details, and performance metrics.
- **System** — ``Device Info``, ``Logs``, and ``Help``.

The home screen itself surfaces the board's connection state and identity at a glance:

- **Connection status** — top-right banner shows ``Connected to AM62D EVM`` along with the
  board's IP address.
- **Info panel** — current ``SDK Version``, ``MCU+ SDK Version``, and ``TIDL Version``
  installed on the board.
- **Device Info panel** — ``Device``, ``Board``, ``SoC`` (AM62Dx quad Cortex-A53 @ 1.4GHz +
  C7x DSP), ``IP Address``, and system ``Uptime``.
- **Runtime Status panel** — live ``CPU Load``, ``CPU Avg``, ``CPU Peak``, and ``RAM
  Used/Free``, refreshed continuously while the portal is open.

Speech Enhancement
==================

.. figure:: /images/am62dx_web_gui_speech_enhancement.png
   :align: center

   AM62DX web GUI — speech enhancement demo

This demonstration showcases real-time speech enhancement. It integrates noise reduction
powered by the C7x DSP with TIDL-accelerated processing to significantly improve speech quality.

Under ``Demo Selection`` on the left, three options are available:

- **Speech Enhancement** — C7x DSP noise reduction + TIDL.
- **Audio Classification** — AI-based audio event classification.
- **Custom Audio Pipeline** — a generic ``gst-launch`` runner with artifact management for
  running custom GStreamer pipelines from the portal.

The ``About`` and ``Model Info`` collapsible panels describe the demo and the underlying
GCRN-based noise-suppression model. The ``Audio Source`` card shows the active input file
(for example ``input_audio.wav`` — 1ch, 16 kHz, 16-bit, 9.77s) with a ``Default`` tag, and a
status indicator (``Ready``, ``Running``, etc.).

To run Speech Enhancement:

1. Select Run Demo to begin the process
2. Allow the system to complete the processing cycle.
3. Once finished, the interface will display both the original input and the processed output audio files.
4. Use the play button on the input file to hear the original noisy audio.
5. Use the play button on the output file to hear the enhanced version.

Results:

The ``Visualization`` section plots both a spectrogram and a waveform for the ``Input
(Noisy)`` and ``Output (Enhanced)`` audio, side by side, with hover tooltips reporting
time, frequency, and spectral level (or sample amplitude for the waveform). Spectrogram
controls let you switch between per-plot peak and a fixed -80 to 0 dB scale, and zoom the
time axis.

.. figure:: /images/am62dx_web_gui_speech_enhancement_result_1.png
   :align: center

   AM62DX web GUI — speech enhancement demo Result 1 (input vs. enhanced spectrograms)


.. figure:: /images/am62dx_web_gui_speech_enhancement_result_2.png
   :align: center

   AM62DX web GUI — speech enhancement demo Result 2 (input vs. enhanced waveforms, and
   per-stage processing times)

Below the waveforms, a ``Processing Times`` table breaks down DSP/TIDL latency per
processed frame range — ``Pre-Processing (STFT)``, ``Model Inference (GCRN)``,
``Post-Processing (ISTFT)``, and ``Total Time`` — useful for verifying real-time
performance on the C7x DSP.


Audio Classification
====================

.. figure:: /images/am62dx_web_gui_audio_classification.png
   :align: center

   AM62DX web GUI — audio classification demo

This demo demonstrates Audio Classification using the YAMNet & VGGish model on the AM62D C7x DSP.

The ``Model`` panel lets you pick the ``Classification Model`` — ``YAMNet`` (521 AudioSet
classes, MobileNet v1) or ``VGGish`` (10 UrbanSound8K classes, VGGish features). The
``Audio Source`` card shows the active input, e.g. ``AM62D2-EVM: pcmdevice-codec-0`` tagged
``Live Microphone`` (live ALSA capture, 16 kHz mono PCM, 1s chunks). The ``Input Source``
dropdown lets you switch between the live microphone and a ``REFRESH`` ed device list, or
``UPLOAD WAV FILE`` to classify a recorded clip instead.

To run the Audio Classification Demo:

1. Select Audio Classification from the Demo Selection menu on the left.
2. Select Model from the model list
3. Select audio source mic/file from dropdown list
4. Click the Run Demo button in the top right corner.
5. Check the audio classification result in output section

While running, a live status line (``Running Audio Classification``) is shown above the
results. The ``Classification Results`` panel lists each detected event with a running
count and timestamp, newest first, and can be cleared with the trash icon. Below it, a
``Top-10 Predictions Over Patches`` heatmap plots the model's top classes against audio
patches over time, with a logit-value color scale for confidence.

YAMNET Model Result:

.. figure:: /images/am62dx_web_gui_audio_classification_yamnet_result.png
   :align: center

   AM62DX web GUI — audio classification YAMNet demo. Classification results list events
   such as ``Silence`` and ``Speech``; the heatmap tracks top-10 classes (Silence, Sound
   effect, Musical instrument, Vehicle, Speech, Inside/large room, Inside/small room,
   Music) across 12 audio patches.

VGGISH Model Result:

.. figure:: /images/am62dx_web_gui_audio_classification_vggish_result.png
   :align: center

   AM62DX web GUI — audio classification VGGish demo, detecting UrbanSound8K classes such
   as ``Engine idling`` from live microphone input.