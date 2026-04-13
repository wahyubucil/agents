---
name: google-dev-knowledge
description: Access the latest Google Developer documentation via MCP. Use when the user asks for up-to-date information on any Google technology (Cloud, Firebase, Android, Angular, Flutter, Maps, AI/ML, etc.), or when official documentation is needed.
---

# Google Developer Knowledge Skill

## Overview

This skill leverages the Google Developer Knowledge Model Context Protocol (MCP) to provide up-to-date information from official Google documentation across the entire Google developer ecosystem.

## Usage

When the user asks about any Google technology or requests official documentation:

1.  **Check for MCP Availability**: Verify if the `google-dev-knowledge` tool (or similar Google documentation tool) is available in the current environment.
2.  **If MCP is Missing**:
    -   Inform the user that the Google Developer Knowledge MCP is required to access the latest documentation.
    -   Provide the link: https://developers.google.com/knowledge/mcp
    -   **STOP** and wait for the user to enable the MCP. Do not attempt to answer using outdated internal knowledge if the user specifically requested up-to-date or official info.
3.  **If MCP is Available**:
    -   Use the MCP tool to search for and retrieve relevant documentation.
    -   Prioritize information returned by the MCP over internal training data, as it is more likely to be current.
    -   Synthesize the information to answer the user's question, citing the source if provided by the MCP.

## Coverage Areas

This skill covers the full Google developer ecosystem, including but not limited to:

-   **Google Cloud Platform (GCP)**: Compute Engine, Cloud Run, GKE, Cloud Functions, IAM, networking, storage, databases, and all GCP services.
-   **Firebase**: Authentication, Firestore, Realtime Database, Cloud Messaging, Hosting, Remote Config, and other Firebase products.
-   **Android**: Jetpack, Compose, Kotlin, Material Design, Play Services, and Android platform APIs.
-   **Web**: Angular, Chrome DevTools, Web APIs, Progressive Web Apps, Lighthouse, and web performance.
-   **Flutter**: Cross-platform development with Flutter and Dart.
-   **AI/ML**: Gemini API, Vertex AI, TensorFlow, MediaPipe, and Google AI tools.
-   **Maps & Location**: Maps SDK, Places API, Routes API, and geolocation services.
-   **Identity & Security**: OAuth, OpenID Connect, Identity Platform, and reCAPTCHA.
-   **Other**: Google Workspace APIs, Ads APIs, YouTube APIs, and any other Google developer products.

## Error Handling

If the MCP tool returns an error or no results:
-   Fall back to internal knowledge but explicitly state that the information might be outdated and suggest checking the official documentation URL directly.
