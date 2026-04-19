# WebCodecs Video Playground (Svelte)

A high-performance technical demonstration of frame-accurate video scrubbing using the WebCodecs API and Svelte.

This project explores bypassing the limitations of the standard HTML5 <video> element to achieve the sub-millisecond seeking precision required for professional non-linear editors (NLEs).

🛠 The Tech Stack
Framework: Svelte 5 / TypeScript

Core API: WebCodecs API (VideoDecoder, EncodedVideoChunk)

Rendering: HTML5 Canvas (Hardware Accelerated)

# Engineering Objectives
The goal of this playground is to demonstrate a "zero-lag" interface for performance-sensitive video applications:

Low-Latency Seeking: Manual control over the decoding pipeline to eliminate the "buffering" delays found in standard browser players.

Memory Management: Implementing strict frame.close() patterns to ensure the browser's GPU memory remains stable during intensive scrubbing.

Frame-to-Pixel Mapping: Precise synchronization between the Svelte-reactive timeline and the Canvas draw loop.

# Key Research & Implementation

Bitstream Demuxing: Architectural planning for extracting elementary streams (AVC/HEVC) for raw decoding.

Retina-Ready Rendering: Handling high-DPI canvas scaling to ensure UI controls remain crisp on 4K/5K displays.

Optimization: Configured the VideoDecoder with optimizeForLatency: true to prioritize immediate visual feedback over background buffering.