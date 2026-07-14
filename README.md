TinyML application that performs real-time Keyword Spotting (KWS) on an ESP32 microcontroller using a highly optimized, quantized 8-bit Integer (Int8) Convolutional Neural Network (CNN) simulation pipeline.

##  Project Overview

Traditional audio processing handles massive time-series signals, which are too resource-intensive for resource-constrained microcontrollers like the ESP32. This project utilizes an optimized **Edge Inference Pipeline** that transforms audio data into a visual representation before classification. 

The system simulates an onboard feature extraction layer, taking a **40x32 spectrogram matrix** and evaluating it against a lightweight internal weight routing matrix to instantly classify spoken keywords (`YES` vs. `NO`) with microscopic execution latency.

---

##  Core Architecture & Technical Concepts

### 1. The Audio-to-Image Pipeline (FFT & Spectrograms)
Microcontrollers cannot easily parse a raw `.wav` file directly. To solve this, the signal is converted into an "image":
* **Fast Fourier Transform (FFT):** Slices a 1-second audio clip into tiny, overlapping 30\text{ ms} time windows (Frames) to mathematically extract the specific pitches and frequencies present.
* **Hanning Window:** Abruptly cutting off an audio wave creates harsh, artificial cuts at frame boundaries, which mathematically manifest as high-frequency clicks and noise. We apply a mathematical smoothing curve (Hanning Window) to act as a fade-in/fade-out effect, gently tapering the data to zero at the edges while keeping the core sound intact.
* **Spectrogram Generation:** By stacking these flat FFT snapshot arrays side-by-side, we construct a 2D image matrix where the **Row (r) represents Pitch/Frequency** and the **Column (c) represents Time**.
  <img width="824" height="394" alt="image" src="https://github.com/user-attachments/assets/25adc5ce-aaed-4593-901c-8b1d646c2de5" />
  Figure : Extracted Spectrogram Feature Map for Keyword "No"


### 2. TinyML Optimization: Quantization
To ensure the model can process streams in real-time (calculating the math faster than a user can finish speaking), the model undergoes **Post-Training Quantization** via TensorFlow Lite / Google Colab. 
* Converts heavy Float32 decimals into simple, tiny 8-bit whole numbers (Int8) bounded between -128 and 127.
* Compresses the network weights down to a native **44.45 KB footprint**, allowing the entire model to run completely within the local cache of the ESP32 chip.
* Massively increases computational efficiency at a negligible cost to predictive accuracy.

---

##  Understanding the Simulated Model Matrix

The standalone simulation tests the performance and layout logic by feeding a pre-constructed frequency grid that mimics real audio patterns:

* **Frequency Matrix Layout:** A 2D grid of `[FREQUENCY_BINS]` \times `[TIME_FRAMES]` (40 \times 32).
* **Keyword Detection Logic:**
  * **`NO` Characterization:** Typically maps high energy to the lower frequency spectrum (r < 8).
   <img width="704" height="152" alt="image" src="https://github.com/user-attachments/assets/e6799861-fba8-498b-96e8-6bf3e1b8eea4" />
   
  * **`YES` Characterization:** Indicated by a high-energy pattern in the higher frequency/hiss bands (r > 25 and 10 < c < 24).
   <img width="762" height="146" alt="image" src="https://github.com/user-attachments/assets/5c5c569b-d69e-46e3-9cea-23a6c4b65ca1" />


###  Why are the simulation confidence scores linear?
In a full TensorFlow deployment, the final layer uses a **Softmax function** to normalize output logits (raw scores). Softmax raises Euler's constant (e \approx 2.718) to the power of each score:
<img width="753" height="163" alt="image" src="https://github.com/user-attachments/assets/fd035fdd-c42b-4c28-b66a-1db382b8cbf3" />


Microcontrollers require significant processing clock cycles to compute floating-point exponents. To maximize edge performance, this standalone C++ engine utilizes a fast, **un-amplified linear ratio** of the accumulators:

\text{Linear Confidence} = \frac{\vert{}A_{\text{target}}\vert{}}{\vert{}A_{\text{no}}\vert{} + \vert{}A_{\text{yes}}\vert{}} \times 100\%

Because the raw accumulated loop variables remain close in linear bounds, the resulting percentage stays humble and polarized compared to an exponential Softmax output.

---

## 🛠️ Project Structure

```text
├── keyword_classifier.ino  # Main Arduino sketch running the edge inference loop
└── keyword_model.h         # Hexadecimal C++ static array containing model weights
```

---

## How to Run?

1. **Structure:** Place `keyword_classifier.ino` and `keyword_model.h` in the same directory.
2. **Flash:** Open the `.ino` file in the Arduino IDE and upload it to your ESP32.
3. **Connect:** Open the Serial Monitor and set the baud rate to **`115200`**.
4. **Run:** Type **`run`** into the terminal input strip and press **Enter** to trigger the simulated pipeline.
