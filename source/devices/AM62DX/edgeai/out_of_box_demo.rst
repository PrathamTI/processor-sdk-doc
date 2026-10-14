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

The left navigation sidebar starts with ``Home``, then groups the rest into three sections:

- **Demos** — ``Audio Intelligence`` (speech enhancement and audio classification) and
  ``DSP Acceleration`` (2D FFT, biquad filter chain, and workload offload via RPMsg-DMA
  compute demos).
- **Tools** — ``AI Model Inspector`` to upload and browse TIDL model inspection reports,
  including architecture, layer details, and performance metrics.
- **System** — ``Device Info``, ``Logs``, and ``Help``.

The main panel opens with a welcome banner — ``SITARA AM62D DSP + Edge AI Portal`` — and a
**GET STARTED** button that jumps straight to the Audio Intelligence demo. Below it, an
**Explore Demos And Tools** panel repeats the same three entry points (Audio Intelligence, DSP
Acceleration, AI Model Inspector) as clickable cards with a one-line description each, so a
first-time visitor does not need to use the sidebar at all to start exploring.

The home screen itself surfaces the board's connection state and identity at a glance:

- **Connection status** — top-right banner shows ``Connected to AM62D EVM`` along with the
  board's IP address.
- **Info panel** — current ``SDK Version``, ``MCU+ SDK Version``, and ``TIDL Version``
  installed on the board.
- **Device Info panel** — ``Device``, ``Board``, ``SoC`` (AM62Dx quad Cortex-A53 @ 1.4GHz +
  C7x DSP), ``IP Address``, and system ``Uptime``.
- **Runtime Status panel** — live ``CPU Load``, ``CPU Avg``, ``CPU Peak``, and ``RAM
  Used/Free``, refreshed continuously while the portal is open.

A status bar at the bottom of every page repeats ``Uptime``, ``CPU Load``, and ``RAM`` for
quick reference while you are deep in a demo page, along with the portal version, build date,
and a link to the full documentation.

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

1. Click **Run Demo** in the top-right corner to begin the process. The status line below the
   Audio Source card changes from ``Ready`` to running, and the button itself becomes
   **Stop Demo**.
2. Allow the system to complete the processing cycle.
3. Once finished, the interface will display both the original input and the processed output audio files.
4. Use the play button on the input file to hear the original noisy audio.
5. Use the play button on the output file to hear the enhanced version.

Results:

The ``Visualization`` section plots both a spectrogram and a waveform for the ``Input
(Noisy)`` and ``Output (Enhanced)`` audio, stacked one above the other, with hover tooltips
reporting time, frequency, and spectral level (or sample amplitude for the waveform).
Spectrogram controls let you switch between per-plot peak and a fixed -80 to 0 dB color scale,
zoom the time axis, and **SAVE ZIP** to download the plotted data for offline analysis.

.. figure:: /images/am62dx_web_gui_speech_enhancement_result_1.png
   :align: center

   AM62DX web GUI — speech enhancement demo Result 1 (input vs. enhanced spectrograms)


.. figure:: /images/am62dx_web_gui_speech_enhancement_result_2.png
   :align: center

   AM62DX web GUI — speech enhancement demo Result 2 (input vs. enhanced waveforms, and
   per-stage processing times)

Below the waveforms, a ``Processing Times`` table breaks down DSP/TIDL latency per
processed frame range (for example ``0-400``, ``301-701``, ``602-1002`` — overlapping
ranges reflecting the overlap-save chunking described in
:ref:`pub_edgeai_am62dx_enhancement_chunking`) into ``Pre-Processing (STFT)``, ``Model
Inference (GCRN)``, ``Post-Processing (ISTFT)``, and ``Total Time``. In a typical run, STFT
and ISTFT each take on the order of 10-20 ms per chunk while the GCRN model inference step
dominates the total, so this table is the quickest way to see where latency budget on the
DSP is actually going.


Audio Classification
====================

.. figure:: /images/am62dx_web_gui_audio_classification.png
   :align: center

   AM62DX web GUI — audio classification demo

This demo demonstrates Audio Classification using the YAMNet & VGGish model on the AM62D C7x DSP.

The ``Model`` panel lets you pick the ``Classification Model`` — ``YAMNet`` (521 AudioSet
classes, MobileNet v1) or ``VGGish`` (10 UrbanSound8K classes, VGGish features); the panel
shows this class-count and backbone caption under whichever model is selected, so you can
confirm the right one before running. Unlike Speech Enhancement, which defaults to a sample
WAV file, Audio Classification defaults to live input: the ``Audio Source`` card shows
``AM62D2-EVM: pcmdevice-codec-0`` tagged ``Live Microphone`` (live ALSA capture, 16 kHz mono
PCM, 1s chunks). The ``Input Source`` dropdown lets you switch to a different captured device
with ``REFRESH``, or classify a recorded clip instead with ``UPLOAD WAV FILE``.

To run the Audio Classification Demo:

1. Select Audio Classification from the Demo Selection menu on the left.
2. Select Model from the model list — YAMNet or VGGish.
3. Select the audio source: leave it on the live microphone, or pick/upload a WAV file.
4. Click **Run Demo** in the top-right corner; the status line shows ``Idle`` before you start.
5. Check the audio classification result in the output section below.

While running, a live status line (``Running Audio Classification``) replaces ``Idle`` above
the results, and the top-right button becomes **Stop Demo**. The ``Classification Results``
panel lists each detected event with a running count and a wall-clock timestamp, newest first
(for example ``13  Silence  02:11:06 pm``), and can be cleared with the trash icon. Below it, a
``Top-10 Predictions Over Patches`` heatmap plots the model's top classes against successive
audio patches over time, with a logit-value color scale for confidence — useful for seeing
which classes the model was close to choosing, not just the winning one.

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