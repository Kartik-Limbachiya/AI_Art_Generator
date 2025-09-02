# 🎨 AI Art Generator

This project is a sophisticated AI Art Generator that transforms simple text prompts into stylized, artistic images. It's a multi-stage pipeline designed as a Final Year Major Project for a B.Sc. in Information Technology.

The core workflow involves three steps:
1.  **Prompt Enhancement:** A user's basic idea is expanded into a rich, detailed prompt using the Google Gemini API.
2.  **Image Generation:** The enhanced prompt is used to generate a high-quality base image with Stable Diffusion.
3.  **Artistic Stylization:** The generated image is then transformed by applying the artistic style of another image using an improved Neural Style Transfer algorithm.



---

## ✨ Features

-   🤖 **AI-Powered Prompt Enhancement**: Leverages the Google Gemini API to automatically add artistic detail, lighting, and composition notes to simple user prompts.
-   🎨 **High-Quality Image Generation**: Utilizes the powerful `runwayml/stable-diffusion-v1-5` model to produce detailed 512x512 images.
-   🎭 **Improved Neural Style Transfer**: Implements a robust Neural Style Transfer algorithm using a VGG19 model to merge the content of the generated image with a unique artistic style.
-   ⚙️ **Robust & Modular**: Built with clear, separated functions for each stage of the pipeline, including error handling and fallback mechanisms for fetching style images.
-   🖥️ **Command-Line Interface**: Easy to run with a simple CLI that guides the user through the process.

---

##  Workflow

The application follows a sequential pipeline to create the final artwork.



1.  **User Input**: The user provides a basic concept (e.g., "a cat in a library").
2.  **Prompt Enhancement**: The prompt is sent to the Gemini API and returns an enhanced version (e.g., "A photorealistic image of a fluffy ginger cat sleeping on a pile of antique books in a dimly lit, cozy library, soft light filtering through a window, detailed, intricate, high quality").
3.  **Image Generation**: The enhanced prompt is fed into Stable Diffusion to generate the `generated.png` content image.
4.  **Stylization**: The generated image undergoes Neural Style Transfer, using a pre-selected artistic image (e.g., Van Gogh's "Starry Night") as the style reference.
5.  **Final Output**: The process saves the final `styled_final.png` and a side-by-side `comparison.png`.

---

## 🖼️ Demo / Example Output

Here is an example of what the pipeline can produce from a simple prompt.

| Original Prompt | Stable Diffusion Output | Final Styled Image |
| :-------------- | :---------------------: | :----------------: |
| "A lighthouse at night" | <img src="path/to/your/generated.png" width="256"> | <img src="path/to/your/styled_final.png" width="256"> |

*(Note: Replace the image paths with your own example outputs after running the script.)*

---

## 🚀 Getting Started

Follow these instructions to set up and run the project on your local machine.

### Prerequisites

-   Python 3.8+
-   `pip` and `git` or use 'google collab'
-   An **NVIDIA GPU** with CUDA installed is strongly recommended for performance.

### Installation

1.  **Clone the repository:**
    ```sh
    git clone https://github.com/Kartik-Limbachiya/AI_Art_Generator.git
    cd ai-art-generator
    ```

2.  **Create a `requirements.txt` file:**
    Create a file named `requirements.txt` and add the following dependencies:
    ```
    torch
    torchvision
    diffusers
    transformers
    accelerate
    safetensors
    requests
    Pillow
    numpy
    python-dotenv
    ```

3.  **Install the dependencies:**
    ```sh
    pip install -r requirements.txt
    ```

4.  **Set up your API Key:**
    You need a Google Gemini API key for prompt enhancement.
    -   Create a file named `.env` in the root of the project directory.
    -   Add your API key to this file:
        ```
        GEMINI_API_KEY="your...secret...key"
        ```
    The script will automatically load this key.

### Usage

Run the main script from your terminal:

```sh
python main.py
