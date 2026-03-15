# OpenSpeedTest

This directory contains the manifests for deploying [OpenSpeedTest](https://openspeedtest.com/).

OpenSpeedTest is a free, open-source HTML5 network speed test tool. It measures download speed, upload speed, and ping/latency without requiring Flash or Java.

## Image

This deployment uses the `openspeedtest/latest` Docker image.

## Networking

This application is exposed two ways:

- **HTTPRoute** (`speedtest.batlab.io`) — routes through the internal gateway for convenient browser access. Note that reverse proxy buffering can affect the accuracy of speed test results.
- **LoadBalancer** (`192.168.101.6:3000`) — direct access bypassing the reverse proxy, recommended for accurate speed test measurements.

## Dependencies

This application has no external dependencies. It is stateless and requires no persistent storage or secrets.
