.. _am62dx_edgeai_adding_new_model:

####################
Adding a New Model
####################

This page provides a high-level overview of the steps required to integrate a new deep learning model into the AM62DX Edge AI system. It serves as a roadmap linking to detailed documentation for each step.

Overview
========

Adding a new model to the AM62DX Edge AI inference chain involves four main components:

1. **Compile the Model** — Prepare and compile your model using TVM
2. **Firmware Changes** — Integrate the model into the DSP firmware signal chain
3. **Linux Application** — Update the Linux application to support the new model
4. **GStreamer Plugin** — Wire the model into the GStreamer pipeline

Each step builds on the previous one. The exact implementation depends on whether your model is a reconstruction-style model (with synthesis/output stage) or an analysis-only model (feature extraction only).

Step 1: Compile the Model
=========================

**Purpose:** Convert your trained model into a TVM-optimized artifact for the C7x DSP.

This step involves:
- Quantization and optimization for the target DSP
- Generating TVM C7x runtime code
- Creating model artifacts (metadata, weights, graphs)

.. note::
   Detailed guidance on model compilation is available in the TVM documentation.
   Once compiled, you will have artifacts such as ``tvm_model.c``, ``deploy_graph.json``, and model weights.

Step 2: Firmware Changes
=========================

**Purpose:** Integrate the model into the DSP firmware so it can be dispatched and executed.

This involves modifying the TISP signal chain framework:
- Register a new ``ModelId`` in the model configuration
- Create model-specific STFT/ISTFT parameter headers
- Implement the signal chain class with pre-graph and post-graph processing
- Wire the model into the dispatcher
- Update build configurations (CCS and Makefile)

For comprehensive step-by-step instructions, see:
:ref:`MCU+ SDK Edge AI Add New Model Guide <https://github.com/TexasInstruments/mcu_plus_sdk/blob/master/docs_src/docs/api_guide_sphinx/examples/signal_processing/edge_ai_add_new_model.md>`

Key files to modify:
- ``common/model_config.h`` — Register the ModelId
- ``common/models/<model>/`` — New model signal chain files
- ``common/signal_chain_dispatch.cpp`` — Register with dispatcher
- Build files (CCS ``.projectspec``, Makefile)

Step 3: Linux Application
=========================

**Purpose:** Update the Linux-side application to configure and run the new model.

The rpmsg inference application needs to:
- Add a new pipeline JSON configuration file for your model
- Define input/output specifications
- Update the application to recognize the new model

For details on configuring pipelines and testing, see:
:ref:`Linux Audio Inference Application <am62dx_linux_audio_inference_app>`

Step 4: GStreamer Plugin
=========================

**Purpose:** Enable the new model to be used via GStreamer pipelines.

The ``titvm`` GStreamer element needs:
- Model metadata in ``deploy_graph.json``
- Plugin element properties for the new model
- GStreamer pipeline examples

For integration details and pipeline examples, see:
:ref:`Audio AI Using GStreamer <pub_edgeai_am62dx_getting_started>`

Summary
=======

The model integration process follows a predictable pattern:

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - Component
     - Key Task
     - Reference
   * - TVM Compiler
     - Compile and optimize model for C7x
     - TVM Documentation
   * - DSP Firmware
     - Implement signal chain and dispatcher wiring
     - MCU+ SDK Edge AI Guide
   * - Linux Application
     - Add pipeline configuration
     - Linux Audio Inference Application
   * - GStreamer
     - Wire into pipeline framework
     - Audio AI Using GStreamer

Each new model follows the same integration pattern, regardless of model architecture. Once you have completed all four steps, your model is ready for deployment and can be selected at runtime.
