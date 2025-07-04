# ComfyUI Audio Batch & Utility Nodes &#x0001F3A7;&#x0001F39B;&#xFE0F;

This repository provides a set of custom nodes for ComfyUI focused on audio batching and common audio processing tasks like
channel conversion and resampling. These nodes are designed to help manage and prepare audio data within your ComfyUI
workflows, especially when dealing with multiple audio inputs or outputs.

## &#x0001F4DC; Table of Contents

- [&#x2728; Nodes](#-nodes)
  - [1. Batch Audios](#1-batch-audios)
  - [2. Select Audio from Batch](#2-select-audio-from-batch)
  - [3. Audio Channel Converter](#3-audio-channel-converter)
  - [4. Audio Force Channels](#4-audio-force-channels)
  - [5. Audio Resampler](#5-audio-resampler)
  - [6. Audio Channel Conv and Resampler](#6-audio-channel-conv-and-resampler)
  - [7. Audio Information](#7-audio-information)
- [&#x0001F680; Installation](#-installation)
- [&#x0001F4E6; Dependencies](#-dependencies)
- [&#x0001F5BC;&#xFE0F; Examples](#&#xFE0F;-examples)
- [&#x0001F4DD; Usage Notes](#-usage-notes)
- [&#x0001F6E0;&#xFE0F; Future Improvements / TODO](#&#xFE0F;-future-improvements--todo)
- [&#x0001F4DC; Project History](#-project-history)
- [&#x2696;&#xFE0F; License](#&#xFE0F;-license)
- [&#x0001F64F; Attributions](#-attributions)

## &#x2728; Nodes

### 1. Batch Audios
   - **Display Name:** `Batch Audios`
   - **Internal Name:** `SET_AudioBatch`
   - **Category:** `audio/batch`
   - **Description:** Takes two audio inputs (which can themselves be batches) and combines them into a single, larger audio batch. The node handles differences in sample rate, channel count, and length between the two inputs to produce a unified batch.
   - **Inputs:**
     - `audio1` (AUDIO): The first audio input. Can be a single audio item or a batch.
     - `audio2` (AUDIO): The second audio input. Can be a single audio item or a batch.
   - **Output:**
     - `audio_batch` (AUDIO): A single audio object where the waveforms from `audio1` and `audio2` are concatenated along the batch dimension.
   - **Behavior Details:**
     - **Sample Rate:** All audio in the output batch will be resampled to match the sample rate of `audio1`.
     - **Channels:**
       - If both inputs are mono, the output batch will be mono.
       - If one input is mono and the other is stereo, the mono input will be converted to "fake stereo" (by duplicating its channel), and the output batch will be stereo.
       - If both inputs are stereo, the output batch will be stereo.
       - For more complex multi-channel inputs, it defaults to the maximum channel count of the two inputs with a warning, as advanced downmixing is not performed.
     - **Length (Samples):** All audio clips in the output batch will be padded with silence at the end to match the length of the longest clip (after any resampling).
     - **Input Batch Handling:** If `audio1` has B1 items and `audio2` has B2 items, the output `audio_batch` will contain B1 + B2 items.

### 2. Select Audio from Batch
   - **Display Name:** `Select Audio from Batch`
   - **Internal Name:** `SET_SelectAudioFromBatch`
   - **Category:** `audio/batch`
   - **Description:** Selects a single audio stream from an input audio batch based on a specified index. Provides options for handling out-of-range indices.
   - **Inputs:**
     - `audio_batch` (AUDIO): An audio batch (e.g., from the "Batch Audios" node).
     - `index` (INT): The 0-based index of the audio stream to select from the batch.
     - `behavior_out_of_range` (COMBO): What to do if the `index` is out of range:
       - `silence_original_length` (default): Output silent audio with the same channel count and duration as items in the original batch.
       - `silence_fixed_length`: Output silent audio with a duration specified by `silence_duration_seconds`.
       - `error`: Raise an error (which will halt the workflow and display an error in ComfyUI).
     - `silence_duration_seconds` (FLOAT): The duration of the silent audio if `behavior_out_of_range` is set to `silence_fixed_length`.
   - **Output:**
     - `selected_audio` (AUDIO): The selected audio stream (as a batch of 1) or silent audio if the index was out of range (and behavior was not "error").

### 3. Audio Channel Converter
   - **Display Name:** `Audio Channel Converter`
   - **Internal Name:** `SET_AudioChannelConverter`
   - **Category:** `audio/conversion`
   - **Description:** Converts the channel layout of an input audio (e.g., mono to stereo, stereo to mono). Handles batches.
   - **Inputs:**
     - `audio` (AUDIO): The input audio.
     - `channel_conversion` (COMBO): The desired channel conversion strategy:
       - `keep` (default): No changes are made to the channel count. Logs a warning if input has more than 2 channels.
       - `stereo_to_mono`: Converts stereo (or multi-channel) audio to mono by averaging all input channels. If already mono, no change.
       - `mono_to_stereo`: Converts mono audio to "fake stereo" by duplicating the mono channel. If already stereo, no change. For multi-channel (>2) inputs, it takes the first channel and duplicates it to create stereo.
       - `force_mono`: Always converts the input to mono by averaging all channels, regardless of the original channel count.
       - `force_stereo`:
         - If input is mono, converts to "fake stereo".
         - If input is stereo, no change.
         - If input has more than 2 channels, it takes the first channel and duplicates it to create stereo.
   - **Output:**
     - `audio_out` (AUDIO): The audio with the converted channel layout. The batch size and sample rate are preserved.

### 4. Audio Force Channels
   - **Display Name:** `Audio Force Channels`
   - **Internal Name:** `SET_AudioForceChannels`
   - **Category:** `audio/conversion`
   - **Description:** Forces the number of channels. 0 means keep same.
   - **Inputs:**
     - `audio` (AUDIO): The input audio.
     - `channels` (INT): The desired channel number of channels. This is equivalent to **Audio Channel Converter**:
       - `0`: `keep`
       - `1`: `force_mono`
       - `2`: `force_stereo`
   - **Output:**
     - `audio` (AUDIO): The audio with the converted channel layout. The batch size and sample rate are preserved.

### 5. Audio Resampler
   - **Display Name:** `Audio Resampler`
   - **Internal Name:** `SET_AudioResampler`
   - **Category:** `audio/conversion`
   - **Description:** Resamples the input audio to a specified target sample rate using `torchaudio.transforms.Resample`. Handles batches.
   - **Inputs:**
     - `audio` (AUDIO): The input audio.
     - `target_sample_rate` (INT): The desired sample rate in Hz (e.g., 44100, 48000, 16000). If set to 0 or if it matches the original sample rate, resampling is skipped.
   - **Output:**
     - `audio_out` (AUDIO): The resampled audio. The batch size and channel count are preserved.

### 6. Audio Channel Conv and Resampler
   - **Display Name:** `Audio Channel Conv and Resampler`
   - **Internal Name:** `SET_AudioChannelConvResampler`
   - **Category:** `audio/conversion`
   - **Description:** A convenience node that combines channel conversion and resampling into a single step.
   - **Inputs:**
     - `audio` (AUDIO): The input audio.
     - `channel_conversion` (COMBO): Same options as the "Audio Channel Converter" node.
     - `target_sample_rate` (INT): Same options as the "Audio Resampler" node.
   - **Output:**
     - `audio_out` (AUDIO): The audio after both channel conversion and resampling have been applied.

### 7. Audio Information
   - **Display Name:** `Audio Information`
   - **Internal Name:** `SET_AudioInfo`
   - **Category:** `audio/conversion`
   - **Description:** Shows information about the audio.
   - **Inputs:**
     - `audio` (AUDIO): The input audio.
   - **Output:**
     - `audio_bypass` (AUDIO): The audio from the input, here to make easier its use.
     - `batch_size` (INT): Size of the audio batch, how many sounds are in the batch.
     - `channels` (INT): Number of audio channels (1 mono, 2 stereo)
     - `num_samples` (INT): How many samples contains the audio. Duratio [s] = `num_samples` / `sample_rate`
     - `sample_rate` (INT): Sampling frequency, how many samples per second.

## &#x0001F680; Installation

You can install the nodes from the ComfyUI nodes manager, the name is *Audio Batch*, or just do it manually:

1.  Clone this repository into your `ComfyUI/custom_nodes/` directory:
    ```bash
    cd ComfyUI/custom_nodes/
    git clone https://github.com/set-soft/ComfyUI-AudioBatch ComfyUI-AudioBatch
    ```
2.  Restart ComfyUI.

The nodes should then appear under the "audio/batch" and "audio/conversion" categories in the "Add Node" menu.

## &#x0001F4E6; Dependencies

- PyTorch
- Torchaudio (for resampling and potentially other audio operations)
- NumPy (often used with audio data)

These are typically already present in a standard ComfyUI environment.

## &#x0001F5BC;&#xFE0F; Examples

Once installed the examples are available in the ComfyUI workflow templates, in the *Audio Batch* section.

- [audio_batch_select_example.json](example_workflows/audio_batch_select_example.json): Shows how to create a batch and
  how to extract a single element from the batch.
- [resample_force_stereo.json](example_workflows/resample_force_stereo.json): Shows how to change the number of channels
  and the sample rate.

## &#x0001F4DD; Usage Notes

- **AUDIO Type:** These nodes work with ComfyUI's standard "AUDIO" data type, which is a Python dictionary containing:
  - `'waveform'`: A `torch.Tensor` of shape `(batch_size, num_channels, num_samples)`.
  - `'sample_rate'`: An `int` representing the sample rate in Hz.
- **Logging:** &#x0001F50A; The nodes use Python's `logging` module. Debug messages can be helpful for understanding the transformations being applied.
  You can control log verbosity through ComfyUI's startup arguments (e.g., `--preview-method auto --verbose DEBUG` for more detailed ComfyUI logs
  which might also affect custom node loggers if they are configured to inherit levels). The logger name used is "AudioBatch".
  You can force debugging level for these nodes defining the `AUDIOBATCH_NODES_DEBUG` environment variable to `1`.

## &#x0001F6E0;&#xFE0F; Future Improvements / TODO

- Add more sophisticated downmixing options for multi-channel audio (e.g., 5.1 to stereo).
- Allow user to choose padding value (e.g., silence, edge, reflect) for length matching in "Batch Audios".
- Option in "Batch Audios" to truncate to shortest instead of padding to longest.
- More options for stereo-to-mono conversion (e.g., take left channel, take right channel).
- If you are interested on them, please open an issue.

## &#x0001F4DC; Project History

- 1.0.0 2025-06-03: Initial release
  - Initial 5 nodes: `Batch Audios`, `Select Audio from Batch`, `Audio Channel Converter`, `Audio Resampler` and `Audio Channel Conv and Resampler`

- 1.1.0 2025-06-30: Two new nodes
  - Added 2 new nodes: `Audio Force Channels` and `Audio Information`

- 1.1.1 2025-06-30: Just better project description

## &#x2696;&#xFE0F; License

[GPL-3.0](LICENSE)

## &#x0001F64F; Attributions

- Good part of the initial code and this README was generated using Gemini 2.5 Pro.
