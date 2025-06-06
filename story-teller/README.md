# Story Teller iOS App Plan

## Overview

Story Teller is an iPhone application that takes a text file containing a short story, analyzes it using the OpenAI ChatGPT API to build a list of characters, assigns each character a distinct voice, and then reads the story aloud with those voices. The app requests an API key from the user and stores it securely in the iOS Keychain. Voice playback is handled through `AVSpeechSynthesizer` using available iOS voices.

This document breaks down development into phases that you can later assign as implementation tasks.

## Phase 1: Xcode Project Setup

1. **Create a new Xcode project** using the "App" template with SwiftUI.
2. **Set up Swift Package Manager dependencies** if needed (e.g., `Alamofire` for networking, although `URLSession` can suffice).
3. **Organize project structure**:
   - `Models` – data objects for stories, characters, and API responses.
   - `Views` – SwiftUI screens for file import, key entry, and playback.
   - `ViewModels` – logic layer to coordinate parsing, API calls, and speech.
4. **Configure basic app settings** (bundle identifier, deployment target, etc.).

## Phase 2: User Input & API Key Handling

1. **Story Import**
   - Use the `DocumentPicker` to let users select a plain-text file from Files.
   - Read the text into a `String` and store it in memory or the app sandbox.
2. **API Key Entry**
   - Create a settings screen where the user pastes their OpenAI API key.
   - Save the key securely in the Keychain using `KeychainAccess` or native APIs.
   - Expose a simple function `func loadAPIKey() -> String?` for the network layer.
3. **Basic UI**
   - Show selected file name and a button to start analysis once a story and API key are present.

## Phase 3: Story Analysis via ChatGPT

1. **Prompt Construction**
   - Craft a prompt instructing ChatGPT to return a JSON list of characters with attributes (name, gender if known, brief description) and a sequence of story segments tagged with the speaking character or "narrator".
   - Example prompt stored in a constant for easy tweaking.
2. **Network Request**
   - Build a `ChatGPTService` that performs a `POST` to `https://api.openai.com/v1/chat/completions`.
   - Set the `Authorization` header to `Bearer <API Key>` retrieved from Keychain.
   - Use `URLSession` with async/await for clarity.
3. **Response Parsing**
   - Decode the returned JSON into Swift structs (e.g., `CharacterInfo`, `Segment`).
   - Handle API errors gracefully (invalid key, rate limits, etc.).

## Phase 4: Voice Assignment

1. **Voice Pool**
   - Query `AVSpeechSynthesisVoice.speechVoices()` to get available system voices.
   - Optionally categorize by language and gender.
2. **Mapping Logic**
   - For each detected character, pick an unused voice matching the character's gender when possible.
   - Persist this mapping for the story so playback can be repeated without reassigning voices.
3. **User Overrides**
   - Provide a simple screen to let users choose a different voice for a character.

## Phase 5: Speech Generation & Playback

1. **Segment Queue**
   - Convert each story segment into an `AVSpeechUtterance` with the assigned voice.
   - Maintain a queue to speak segments sequentially.
2. **Playback Controls**
   - Build a basic player interface with play/pause, progress bar, and skip controls.
   - Use `AVSpeechSynthesizerDelegate` to update UI as each utterance finishes.
3. **Error Handling**
   - If speech synthesis fails for a voice, fall back to a default voice and log the issue.

## Phase 6: Persistence & Offline Use

1. **Store Parsed Results**
   - Save the analyzed character list and segment data in `FileManager` so the story doesn't need re-analysis on next launch.
2. **Offline Playback**
   - Because synthesis is done locally, previously analyzed stories can be played without an internet connection.
3. **Settings Backup**
   - Offer an option to export/import the API key and story data via iCloud Drive.

## Phase 7: Polishing & App Store Prep

1. **UI/UX Polish**
   - Add onboarding explaining the need for an API key.
   - Provide icons and animations for a friendly experience.
2. **Testing**
   - Unit test the `ChatGPTService` and text parsing logic using sample files.
   - UI test the import and playback flow on multiple devices.
3. **App Store Submission**
   - Prepare screenshots and localized descriptions.
   - Ensure compliance with OpenAI API usage policies.

---

Following this plan should make implementation smoother. Each phase can be tackled independently, gradually building a full-fledged iPhone story reader that leverages ChatGPT for character detection and voice assignment.
