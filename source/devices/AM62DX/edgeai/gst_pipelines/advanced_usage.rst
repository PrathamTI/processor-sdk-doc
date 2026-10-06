.. _pub_edgeai_am62dx_advanced_usage:

================
Advanced Usage
================

Batch Processing a Directory
============================

The enhancement pipeline needs ``max-stream-samples`` set to the padded length of each file.
This script computes it for each WAV file in a directory. It assumes 16 kHz mono 16-bit PCM
WAV files with a 44-byte header, which is what the sample files use:

.. code-block:: bash

   #!/bin/bash
   # Enhance every 16 kHz mono S16LE WAV file in IN_DIR and write the results to OUT_DIR.

   IN_DIR=/data/audio
   OUT_DIR=/data/enhanced
   MODEL=/usr/share/tvm_inference/artifacts/gcrn
   FRAMES=401
   OVERLAP=100
   HOP=160
   mkdir -p "$OUT_DIR"

   for f in "$IN_DIR"/*.wav; do
     name=$(basename "$f" .wav)
     bytes=$(stat -c %s "$f")
     n=$(( (bytes - 44) / 2 ))
     chunk=$(( FRAMES * HOP ))
     step=$(( (FRAMES - OVERLAP) * HOP ))
     max=$(awk -v n="$n" -v c="$chunk" -v s="$step" \
       'BEGIN { k = (n > c) ? int((n - c + s - 1) / s) + 1 : 1; print (k - 1) * s + c }')

     gst-launch-1.0 -q filesrc location="$f" ! wavparse ! \
       audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
       tidspkernel name=stft msg-type=0x1020 model-path=$MODEL \
         hop-size=$HOP fft-size=320 window-frames=$FRAMES batch-size=64 \
         max-stream-samples=$max ! \
       tidspkernel name=deinterleave msg-type=0x1040 interleave_direction=0 \
         model-path=$MODEL fft-size=320 window-frames=$FRAMES ! \
       titvm model-path=$MODEL ! \
       tidspkernel name=interleave msg-type=0x1040 interleave_direction=1 \
         model-path=$MODEL fft-size=320 window-frames=$FRAMES ! \
       tidspkernel name=istft msg-type=0x1030 model-path=$MODEL \
         hop-size=$HOP fft-size=320 window-frames=$FRAMES batch-size=64 \
         max-stream-samples=$max ! \
       audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
       wavenc ! filesink location="$OUT_DIR/${name}_enhanced.wav"
   done

This script has not been run on the board. Files shorter than one window (401 frames, about
4 seconds) use the same formula, but that case has not been tested either. Check the output
for those files.

Input Format Conversion
=======================

If the source is not 16 kHz mono 16-bit PCM, convert it inside the pipeline before the
``audio/x-raw`` capsfilter:

.. code-block:: bash

   filesrc location=input_44k.wav ! wavparse ! \
     audioconvert ! audioresample ! \
     audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     ... rest of the pipeline

Or convert the file once with ffmpeg:

.. code-block:: bash

   ffmpeg -i input_44k.wav -ar 16000 -ac 1 -sample_fmt s16 input_16k.wav

Running Classification and Enhancement Together
===============================================

A ``tee`` can send the same input to the enhancement chain and to a classifier. Each branch
needs its own ``queue``. This arrangement has not been run on the board.

.. code-block:: bash

   gst-launch-1.0 -v filesrc location=$INPUT ! wavparse ! \
     audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tee name=t \
     t. ! queue ! tidspkernel name=stft_y msg-type=0x1020 \
       model-path=/usr/share/tvm_inference/artifacts/yamnet \
       hop-size=160 window-frames=96 batch-size=64 ! \
     titvm model-path=/usr/share/tvm_inference/artifacts/yamnet \
       class-map-path=/usr/share/tvm_inference/labels/yamnet_label_list.txt top-k=3 ! \
     fakesink \
     t. ! queue ! ... enhancement chain from audio_enhancement ... ! \
     wavenc ! filesink location=/tmp/gst_enhanced.wav

Inside one pipeline, the ``tidspkernel`` elements share one RPMsg channel, and the channel
lock serialises their requests. Two separate pipeline processes each open their own channel.
Running them at the same time has not been tested.

Bring-Your-Own Model
====================

Use this section when you want to run a model other than GCRN, YAMNet or VGGish.

What Works Without Plugin Changes
---------------------------------

- ``titvm`` loads any directory with ``deploy_graph.json``, ``deploy_lib.so`` and
  ``deploy_param.params``. Input and output shapes come from the graph. There are no
  model-name checks in ``titvm``.
- ``titvm`` sends its input to the model daemon when the daemon is running, so a new model
  can be loaded through the daemon in the same way.

What Needs Care
---------------

The DSP stages (``tidspkernel``) are not model-agnostic:

- ``selected-model`` is the firmware ModelId. The firmware implements the spectral processing
  for each ModelId. The plugin does not check that the ModelId matches your model.
- ``model-elems`` must equal the number of spectral values per frame that your model expects.
  The plugin does not read the graph to check it.
- A ``model-path`` that matches no known name does not log a warning. The element keeps its
  defaults (``selected-model=2``, ``model-elems`` from ``fft-size``). Set ``selected-model``
  and ``model-elems`` explicitly on every ``tidspkernel`` element.

Example
-------

For a retrained GCRN model in a different directory, the model must still match the GCRN
spectral layout, and you should keep ModelId 2:

.. code-block:: bash

   MODEL=/usr/share/tvm_inference/artifacts/gcrn_v2
   gst-launch-1.0 filesrc location=$INPUT ! wavparse ! \
     audio/x-raw,format=S16LE,rate=16000,channels=1 ! \
     tidspkernel name=stft msg-type=0x1020 model-path=$MODEL \
       selected-model=2 model-elems=322 hop-size=160 fft-size=320 \
       window-frames=401 batch-size=64 max-stream-samples=160480 ! \
     ... deinterleave, titvm, interleave, istft as in audio_enhancement ...

For a model with a different spectral layout or DSP processing, a new ModelId is needed in the
DSP firmware. That change is outside this repository.

Validation
----------

Before you rely on a new model:

1. Run STFT followed by ISTFT with no model in between. Check that the output reproduces the
   input audio.
2. Run the full chain on a known test file. Compare the output with a reference from the
   model's own pipeline. Checking only the output length is not enough.

Debugging
=========

.. code-block:: bash

   GST_DEBUG=tidspkernel:6,titvm:6,tirpmsgchan:6 gst-launch-1.0 -v ...

To write a pipeline graph:

.. code-block:: bash

   mkdir -p /tmp/dot
   GST_DEBUG_DUMP_DOT_DIR=/tmp/dot gst-launch-1.0 ...
   ls /tmp/dot

The graph files are written when the pipeline changes state. Convert them with ``dot`` on a
host machine if the board does not have it.
