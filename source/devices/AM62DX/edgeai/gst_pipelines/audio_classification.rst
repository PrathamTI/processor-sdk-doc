.. _pub_edgeai_am62dx_audio_classification:

=======================
Audio Classification
=======================

The classification pipelines run YAMNet or VGGish on a 16 kHz mono WAV file and print the
top-k class names and scores for each window to the console.

Models
======

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Model
     - Classes
     - Label file
   * - YAMNet
     - 521
     - ``/usr/share/tvm_inference/labels/yamnet_label_list.txt``
   * - VGGish
     - 10
     - ``/usr/share/tvm_inference/labels/vggish_label_list.txt``

The VGGish label file contains: Air conditioner, Car horn, Children playing, Dog bark,
Drilling, Engine idling, Gun shot, Jackhammer, Siren, Street music.

The YAMNet label file starts with ``Speech`` (line 1), ``Child speech, kid speaking`` (line 2)
and includes ``Dog`` at line 70.

Pipeline Stages
===============

.. code-block:: text

   filesrc → wavparse → audio/x-raw (S16LE, 16 kHz, mono)
     → tidspkernel name=stft msg-type=0x1020   STFT on the DSP
     → titvm model-path=.../yamnet (or vggish) model
     → fakesink

The STFT output goes straight to ``titvm``. There is no separate mel-spectrogram element in
these pipelines. If a model needs mel features, that processing must be part of the model
graph. This has not been checked against the YAMNet and VGGish graphs.

Commands
========

YAMNet
------

.. code-block:: bash

   gst-launch-1.0 -v filesrc location=/usr/share/tvm_inference/input/urbansound_16k.wav ! \
     wavparse ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tidspkernel name=stft msg-type=0x1020 \
       model-path=/usr/share/tvm_inference/artifacts/yamnet \
       hop-size=160 window-frames=96 batch-size=64 ! \
     titvm model-path=/usr/share/tvm_inference/artifacts/yamnet \
       class-map-path=/usr/share/tvm_inference/labels/yamnet_label_list.txt \
       top-k=3 ! \
     fakesink

VGGish
------

.. code-block:: bash

   gst-launch-1.0 -v filesrc location=/usr/share/tvm_inference/input/urbansound_16k.wav ! \
     wavparse ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tidspkernel name=stft msg-type=0x1020 \
       model-path=/usr/share/tvm_inference/artifacts/vggish \
       hop-size=160 window-frames=126 batch-size=64 ! \
     titvm model-path=/usr/share/tvm_inference/artifacts/vggish \
       class-map-path=/usr/share/tvm_inference/labels/vggish_label_list.txt \
       top-k=3 ! \
     fakesink

Both commands omit ``fft-size``. The STFT request does not send an FFT size to the DSP, and
the known-model table sets ``selected-model`` and ``model-elems`` from the path name
(``yamnet`` → ModelId 4, 64 elements per frame; ``vggish`` → ModelId 3, 64 elements per frame).

STFT Settings
=============

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 40

   * - Model
     - ``hop-size``
     - ``window-frames``
     - ``batch-size``
   * - YAMNet
     - 160
     - 96
     - 64
   * - VGGish
     - 160
     - 126
     - 64

``window-frames`` is at most 256 for both models, so plain windowing is used and
``max-stream-samples`` is not needed.

Output Format
=============

When ``class-map-path`` is set, ``titvm`` prints a banner when the model loads, then one block
per window:

.. code-block:: text

   [TVM] Live top-3 prediction printing enabled (521 classes from /usr/share/tvm_inference/labels/yamnet_label_list.txt)
     Top-3 predictions for window 1:
       1. <class name>: <score>
       2. <class name>: <score>
       3. <class name>: <score>

The class count in the banner comes from the label file (521 for YAMNet, 10 for VGGish).
Scores are printed exactly as the model graph produces them. Whether they are probabilities
or logits depends on the model graph and has not been checked for these two models.

Options
=======

``top-k``
  Number of predictions printed per window. Range 1 to 521, default 3.

``class-map-path``
  Label file. If empty, ``titvm`` does not print predictions. The file can be one class name
  per line, or YAML with ``name:`` lines.

Running the Pipeline
====================

1. Check the input, model and labels:

   .. code-block:: bash

      ls /usr/share/tvm_inference/input/urbansound_16k.wav
      ls /usr/share/tvm_inference/artifacts/yamnet/
      ls /usr/share/tvm_inference/labels/yamnet_label_list.txt

2. Run one of the commands above. Use ``-v`` to see the caps negotiation.

3. Read the predictions from the console. Use ``fakesink`` as the last element, since the
   output is the console log rather than a file.

Troubleshooting
===============

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Symptom
     - Action
   * - ``Failed to load class-map-path``
     - Check the path, and that the file is readable.
   * - Banner prints but no predictions appear
     - Run with ``GST_DEBUG=titvm:5``. Check that the pipeline reached PLAYING.
   * - Predictions do not match the expected labels
     - Check that the model and label file are a pair: YAMNet with ``yamnet_label_list.txt``,
       VGGish with ``vggish_label_list.txt``.
   * - ``top-k`` rejected
     - Use a value from 1 to 521.
   * - Long wait before the banner
     - ``titvm`` is waiting for the model daemon. See :ref:`pub_edgeai_am62dx_titvm_execution`.
