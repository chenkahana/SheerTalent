**Project Overview**

This document outlines a detailed, phase-by-phase plan for building an iPhone app (Swift) that:

1. Accepts a story as a text file.
2. Parses the story to extract a list of characters and relevant metadata.
3. Decides on a distinct TTS (text-to-speech) voice for each character, based on their metadata.
4. Reads the story aloud, switching voices seamlessly between characters.
5. Leverages the user’s own ChatGPT API key (entered in-app) to perform character extraction and voice-selection logic.

Below, you’ll find:

* **Section A:** High-level architecture & tech stack
* **Section B:** Data models & core components
* **Section C:** A breakdown into implementation phases (1 through 7), each with in-depth details.

---

## A. Architecture & Tech Stack

1. **Platform & Language**

   * iOS (iPhone) app, target iOS 15+
   * Written in Swift (5.x) with SwiftUI as the UI layer

2. **Key Frameworks & Libraries**

   * **SwiftUI** for UI: file picker, forms, lists, playback controls
   * **Combine** / **async-await** for asynchronous workflows (API calls, file I/O)
   * **AVFoundation (AVSpeechSynthesizer)** for text-to-speech
   * **UniformTypeIdentifiers** & **FileImporter** for reading text files
   * **Keychain Services** (via `KeychainAccess` or similar) to securely store the user’s ChatGPT API key
   * **URLSession** (or a lightweight wrapper) for ChatGPT API calls
   * **Codable** for JSON serialization of ChatGPT responses
   * **Local storage** (e.g., `UserDefaults` or a local JSON file) to cache “indexed” story metadata and user preferences

3. **External Service**

   * OpenAI ChatGPT REST API (v1). The user provides their own API key at runtime.

4. **High-Level Data Flow**

   ```
   ┌──────────┐       1       ┌──────────────────┐
   │ User      │──selects──▶│ Story Loader      │
   │ Interface │◀─file──────│ (FileImporter)    │
   └──────────┘              └──────────────────┘
                                     │
                                     │ 2. rawText (String)
                                     ▼
                          ┌─────────────────────────┐
                          │ Character Extraction     │
                          │ (ChatGPT API call       │
                          │  → JSON of characters)  │
                          └─────────────────────────┘
                                     │
                                     │ 3. [Character]
                                     ▼
                          ┌─────────────────────────┐
                          │ Voice Selection          │
                          │ (ChatGPT or local logic) │
                          └─────────────────────────┘
                                     │
                                     │ 4. [Character + VoiceConfig]
                                     ▼
                          ┌─────────────────────────┐
                          │ Story Annotator          │
                          │ (Optionally splits text  │
                          │  into “speaking turns”)  │
                          └─────────────────────────┘
                                     │
                                     │ 5. [AnnotatedSegments]
                                     ▼
                          ┌─────────────────────────┐
                          │ Playback Engine          │
                          │ (AVSpeechSynthesizer     │
                          │  → dynamic voice)        │
                          └─────────────────────────┘
                                     │
                                     │ 6. Audio
                                     ▼
                           ┌─────────────────────┐
                           │ Speaker (User)      │
                           └─────────────────────┘
   ```

---

## B. Data Models & Core Components

### 1. Data Models

1. **`Story`**

   ```swift
   struct Story {
       let id: UUID
       let title: String
       let rawText: String
       var characters: [Character]
       var segments: [StorySegment]   // (see below)
       var dateIndexed: Date
   }
   ```

   * `id`: unique identifier per story
   * `title`: derived from filename or first line
   * `rawText`: entire story as one String
   * `characters`: array of `Character` structs, populated after extraction
   * `segments`: array of smaller chunks (e.g., dialogue lines or paragraphs) annotated with a `speakerID`
   * `dateIndexed`: timestamp when this story was “indexed”

2. **`Character`**

   ```swift
   struct Character: Identifiable, Codable {
       let id: UUID
       let name: String
       let attributes: CharacterAttributes
       var voiceConfig: VoiceConfig?
   }
   ```

   * `id`: unique for cross-referencing segments
   * `name`: exactly as ChatGPT returns it (e.g., “Alice”, “Bob Jr.”)
   * `attributes`: see below
   * `voiceConfig`: optional struct that holds the chosen TTS voice parameters

3. **`CharacterAttributes`**

   ```swift
   struct CharacterAttributes: Codable {
       let gender: String?        // “male”, “female”, “nonbinary”, or nil
       let ageRange: String?      // e.g., “adult”, “teenager”, “child”
       let accent: String?        // e.g., “British”, “Southern American”
       let personality: String?   // e.g., “stern”, “cheerful”
       // any other metadata ChatGPT returns
   }
   ```

4. **`VoiceConfig`**

   ```swift
   struct VoiceConfig: Codable {
       let voiceIdentifier: String  // e.g., AVSpeechSynthesisVoice identifier
       let rate: Float              // speaking rate
       let pitchMultiplier: Float   // pitch adjustment
       let volume: Float            // volume (0.0–1.0)
   }
   ```

   * Mapping of attributes → actual AVSpeechSynthesisVoice chosen
   * Default values can be tweaked or overridden by user in UI

5. **`StorySegment`**

   ```swift
   struct StorySegment: Identifiable {
       let id: UUID
       let speakerID: UUID?     // references Character.id; nil if narration
       let text: String         // exactly what will be fed to TTS
   }
   ```

   * Splits the raw story into logical “lines” or “paragraphs,” each associated with a speaker or “nil” for narration
   * The “speaker detection” can be done either by:

     * (A) prompting ChatGPT to return an array of `{speakerName, text}` pairs,
     * (B) heuristic splitting (e.g., “Alice: ‘…’”), then mapping name→Character.id

6. **`UserSettings`**

   ```swift
   struct UserSettings: Codable {
       var chatGPTApiKey: String?
       var defaultVoiceRate: Float
       var defaultVoicePitch: Float
       var defaultVoiceVolume: Float
   }
   ```

   * Stored securely in Keychain + persisted in `UserDefaults` for UI defaults

---

## C. Implementation Phases

Below is a recommended breakdown into phases. Each phase can be treated as a standalone “milestone” that yields a deliverable (and associated tests). Feel free to assign these phases individually.

---

### Phase 1: Project Setup & Basic Infrastructure

#### 1.1. Xcode Project & Swift Package Structure

* Create a new Xcode project named `StoryReader` (App template, SwiftUI, iOS 15+).
* Enable Swift Concurrency (async/await) and ensure Combine is available.
* Create Swift packages (or local modules) for:

  * `Models` (all the `struct`s above)
  * `Services` (ChatGPT API client, File I/O, TTS)
  * `Views` (SwiftUI screens)
  * `ViewModels` (MVVM architecture)

#### 1.2. Dependency Management

* Add any third-party dependencies via Swift Package Manager (SPM):

  * **KeychainAccess** (or similar) for secure storage of API key.
  * Optionally, **Alamofire** if you prefer a network wrapper over URLSession (but URLSession + async/await is sufficient).

#### 1.3. Global Constants & Utilities

* Define `Constants.swift` containing:

  * Base URL for ChatGPT API (e.g., `https://api.openai.com/v1/chat/completions`)
  * Supported file extensions (`.txt`, `.rtf` if desired)
  * Default AVSpeechSynthesizer parameters
* Create a `Logger.swift` (lightweight wrapper around `os.log` or simply `print`) for debugging.

#### 1.4. UserSettings & Keychain Integration

* Wrap Keychain calls in a `KeychainManager` class to securely save/retrieve the ChatGPT API key.
* Implement `UserSettings` as a `@AppStorage`-backed (or Combine-backed) singleton that reads/writes to `UserDefaults`, except for the API key, which goes in Keychain.

#### 1.5. Basic App Shell UI

* **Landing Screen** (if no API key stored):

  * TextField for “Enter ChatGPT API Key”
  * Button: “Save & Continue” (validates non-empty)
  * On save: store key in Keychain, transition to Main Screen
* **Main Screen** (if API key already present):

  * Sidebar (iPad) or TabView (iPhone) with two tabs:

    1. **Stories** (list of imported stories)
    2. **Settings** (manage API key, default voice params)

---

### Phase 2: File Import & Local Storage of Raw Story

#### 2.1. File Picker Integration

* On “Stories” tab, provide a “+ Import Story” button.
* Present a `DocumentPicker` (via SwiftUI’s `fileImporter`) restricted to `.txt` files.
* Upon user selection:

  1. Read file’s `URL`.
  2. Use `FileManager` or `String(contentsOf:)` to read raw text into memory (UTF-8).
  3. Derive a “title” from the filename (strip extension).
  4. Instantiate a new `Story` object with `rawText` and placeholder values for `characters` & `segments`.
  5. Save this `Story` into a local “Stories” directory in App’s sandbox (e.g., `Documents/stories/<UUID>.json`).

#### 2.2. Local Persistence (Caching)

* Implement a small file-based persistence layer (`StoryStorageManager`):

  * **Save**: Accept a `Story` instance, encode to JSON (`Codable`), write to disk under `Documents/stories/<story.id>.json`.
  * **Load**: On app launch, scan `Documents/stories` folder, decode each JSON into `Story` structs, populate an in-memory array.
  * **Delete/Rename**: Provide functionality to remove or rename a story.

#### 2.3. UI: Stories List

* Display a SwiftUI `List` of all stored stories (showing `title` and `dateIndexed` if available).
* Tapping a story navigates to the “Story Detail” screen (initially, show “Not yet indexed” if `characters.isEmpty`).

---

### Phase 3: Character Extraction via ChatGPT

#### 3.1. Define ChatGPT Client Service

* Create `ChatGPTService.swift` in `Services/`:

  * Expose a method:

    ```swift
    func extractCharacters(from storyText: String) async throws -> [Character]
    ```
  * Inside, construct the proper HTTP request to OpenAI:

    * **Endpoint:** `POST https://api.openai.com/v1/chat/completions`
    * **Headers:**

      * `Authorization: Bearer <apiKey>` (fetched from Keychain)
      * `Content-Type: application/json`
    * **Payload:**

      ```jsonc
      {
        "model": "gpt-4",   // or “gpt-3.5-turbo” if the user doesn’t have gpt-4
        "messages": [
          {
            "role": "system",
            "content": "You are a smart assistant that receives a story text and outputs a JSON array of characters. Each character object should have: name, gender (male/female/other), ageRange (e.g., child, teenager, adult), accent (if obvious), personality (concise label). Respond only with valid JSON—an array of objects."
          },
          {
            "role": "user",
            "content": "<entire story text here>"
          }
        ],
        "temperature": 0.2
      }
      ```
    * Use `URLSession.shared.data(for: request)` with async/await.
    * On success: decode `response.data` → custom `struct ChatGPTCharacterResponse: Codable { let name: String; let gender: String?; ... }[]`.
    * Then map each returned object to our `Character(id: UUID(), name:…, attributes:…)`.
    * **Error handling:**

      * 401 → invalid API key → propagate a custom `ChatGPTError.invalidAPIKey`
      * timeouts → retry once, then fail
      * malformed JSON → provide fallback: empty array + show “Could not parse character list.”

#### 3.2. Prompt Engineering & Testing

* Iteratively refine the “system” prompt to ensure ChatGPT consistently returns:

  ```jsonc
  [
    {
      "name": "Alice",
      "gender": "female",
      "ageRange": "adult",
      "accent": "British",
      "personality": "curious"
    },
    ...
  ]
  ```
* Write unit tests (mocking URLSession) to verify that given a sample story, the JSON is correctly parsed into `[Character]`.

#### 3.3. UI: “Index Story” Button

* On “Story Detail” screen: show a button “Index Story” if `characters.isEmpty`.
* When tapped:

  1. Disable button, show a `ProgressView("Extracting characters…")`.
  2. Call `ChatGPTService.extractCharacters(from: story.rawText)`.
  3. On return, assign `story.characters = extractedCharacters`.
  4. Persist the updated `Story` (via `StoryStorageManager.save(story)`).
  5. Transition to the next sub-view (voice selection) or allow user to review the character list.
  6. If error, show an alert with a retry option (e.g., “Invalid API Key—please update in Settings.”).

#### 3.4. UI: Displaying Character List

* After successful extraction, display a `List` of all `Character.name` along with a summary of their attributes underneath (e.g., “Female, adult, British, curious”).
* Each row navigates to a “Character Detail” sub-screen where the user can override/edit attributes if desired (optional future enhancement).

---

### Phase 4: Voice Selection & Configuration

#### 4.1. VoiceConfig Generation

##### Option A: Automated Voice Suggestion via ChatGPT

1. **Service Method**

   ```swift
   func suggestVoice(for character: Character) async throws -> VoiceConfig
   ```

   * Send a prompt to ChatGPT such as:

     ```jsonc
     {
       "model": "gpt-4",
       "messages": [
         {
           "role": "system",
           "content": "You are an assistant that suggests an AVSpeechSynthesisVoice identifier and basic TTS settings for a character. Input: name, gender, ageRange, accent, personality. Output JSON: { \"voiceIdentifier\": <string>, \"rate\": <number>, \"pitchMultiplier\": <number>, \"volume\": <number> }. Choose a voice that matches the attributes. Use only Apple’s built-in iOS voice identifiers (e.g., ‘com.apple.ttsbundle.Samantha-compact’, etc.)."
         },
         {
           "role": "user",
           "content": "{ \"name\": \"Alice\", \"gender\": \"female\", \"ageRange\": \"adult\", \"accent\": \"British\", \"personality\": \"curious\" }"
         }
       ],
       "temperature": 0.3
     }
     ```
   * Parse ChatGPT’s JSON response into `VoiceConfig`.
   * **Note:** In practice, ChatGPT might suggest voices that are outdated or mis-named. To mitigate:

     * Include in the system prompt a **reference list** of valid voice identifiers (fetched at runtime from `AVSpeechSynthesisVoice.speechVoices()`).
     * E.g., “Here are the valid voice identifiers on iOS 15: \[list]. Only choose from these.”
   * If ChatGPT returns an invalid identifier, fallback to a heuristic (e.g., map `gender=“female”` → AVSpeechSynthesisVoice(language: “en-US”) with `voiceIdentifier` = `com.apple.ttsbundle.Samantha-compact`).

##### Option B: Local Heuristic Voice Mapping

* As an alternative (or fallback), define a local mapping table:

  ```swift
  let localVoiceMapping: [ (gender: String, ageRange: String, accent: String?) : (voiceID: String, rate: Float, pitch: Float ) ] = [
      ( "male",   "adult",   nil ) : ( "com.apple.ttsbundle.Daniel-compact", 0.45, 1.0 ),
      ( "female", "adult",   nil ) : ( "com.apple.ttsbundle.Samantha-compact", 0.50, 1.0 ),
      ( "male",   "teenager",nil ) : ( "com.apple.ttsbundle.Alex-compact", 0.55, 1.05),
      // etc.
  ]
  ```
* If ChatGPT is unavailable or if user opts out, use this local table.

#### 4.2. Bulk Voice Assignment

* In “Story Detail” screen, add a button “Assign Voices Automatically.”

  * When tapped: iterate over `story.characters`, calling `suggestVoice(for:)` (or local heuristic).
  * Update each `character.voiceConfig` in memory.
  * Persist the `Story` again.
  * Show a summary list: “Alice → Samantha (British, rate 0.50)… Bob → Alex (American)… etc.”

#### 4.3. Manual Override UI

* On “Character Detail” screen (tap from character list), show:

  * A `Picker` of available `AVSpeechSynthesisVoice.speechVoices()` (display `voice.name` in the list).
  * Sliders for `rate` (0.0–1.0), `pitchMultiplier` (0.5–2.0), `volume` (0.0–1.0).
  * Preview button: “▶ Preview Sample.” When tapped, run a short TTS line (e.g., “Hello, I’m \<character.name>”) with the selected voiceConfig.
  * “Save” button to store edits back into `story.characters`.
  * “Restore Default” to revert to what the ChatGPT suggestion or heuristic provided.

---

### Phase 5: Story Segmentation & Annotation

#### 5.1. Annotating the Story with Speaker Tags

##### Approach 1: ChatGPT-Assisted Segmentation

* Extend `ChatGPTService` with:

  ```swift
  func annotateStory(_ text: String, using characters: [Character]) async throws -> [StorySegment]
  ```

  * Construct a prompt:

    ```jsonc
    {
      "model": "gpt-4",
      "messages": [
        {
          "role": "system",
          "content": "You are an assistant that takes a story’s raw text and a list of character names, and outputs an array of objects: { \"speaker\": <characterName or null for narration>, \"text\": <string> }. Each object represents a minimal speaking segment. Respond only in JSON."
        },
        {
          "role": "user",
          "content": "{ \"story\": \"<full story text here>\", \"characters\": [\"Alice\",\"Bob\",\"Charlie\"] }"
        }
      ],
      "temperature": 0.1
    }
    ```
  * Parse the JSON into an intermediate `[ ChatGPTRawSegment ]`, where `ChatGPTRawSegment` is `{ speaker: String?; text: String }`.
  * Map each `speakerName` to the corresponding `UUID` in `Character` (or `nil` if speaker = `null` or “Narrator”).
  * Instantiate `[StorySegment]` with new `UUID()` for each.

##### Approach 2: Local Heuristic Segmentation

* If the story is formatted with explicit speaker prefixes (e.g., `Alice: “…”`), implement a simple parser:

  1. Split `rawText` by newlines.
  2. For each line, use a regex like `^([A-Za-z0-9 ]+):\s*(.*)$` to detect `speakerName: dialogue`.
  3. If matched, find `character.id` for `speakerName`; else treat the entire line as “narration.”
  4. Merge consecutive narration lines into one segment for efficiency.

  * **Drawback:** Doesn’t handle block quotes or paragraphs that alternate lines without explicit prefixes.

#### 5.2. Caching Segmented Output

* Once annotated, attach `[StorySegment]` to `story.segments` and persist.
* Store a timestamp for when annotation occurred, so we don’t re-annotate unless the user explicitly requests.

#### 5.3. UI: Segmentation Status & Preview

* On “Story Detail,” show:

  * If `segments` is empty: a button “Annotate Story for Playback.”
  * If not empty: show a collapsed summary (e.g., “200 segments created.”) and a “Re-Annotate” button (in case story changed or fields edited).
* Optionally, display a `List` of the first 5–10 segments:

  * “Alice: ‘…’”
  * “Bob: ‘…’”
  * “(Narration): ‘….’”

---

### Phase 6: Playback Engine

#### 6.1. AVSpeechSynthesizer Setup

* Create a singleton `SpeechManager` class:

  ```swift
  class SpeechManager: ObservableObject {
      static let shared = SpeechManager()
      private let synthesizer = AVSpeechSynthesizer()
      @Published var isSpeaking: Bool = false
      @Published var currentSegmentIndex: Int?
      // ...
  }
  ```
* Conform to `AVSpeechSynthesizerDelegate` to track progress:

  * `speechSynthesizer(_:didStart:)`
  * `speechSynthesizer(_:didFinish:)`
  * `speechSynthesizer(_:didCancel:)`
  * Set `isSpeaking = true/false` accordingly.
  * When a segment finishes, automatically queue the next segment.

#### 6.2. Sequential Playback of StorySegments

* In `SpeechManager`, implement:

  ```swift
  func play(story: Story) {
      guard !story.segments.isEmpty else { return }
      currentSegmentIndex = 0
      speak(segment: story.segments[0], in: story)
  }

  private func speak(segment: StorySegment, in story: Story) {
      let utterance = AVSpeechUtterance(string: segment.text)
      if let speakerID = segment.speakerID,
         let character = story.characters.first(where: { $0.id == speakerID }),
         let voiceConfig = character.voiceConfig,
         let voice = AVSpeechSynthesisVoice(identifier: voiceConfig.voiceIdentifier) 
      {
          utterance.voice = voice
          utterance.rate = voiceConfig.rate
          utterance.pitchMultiplier = voiceConfig.pitchMultiplier
          utterance.volume = voiceConfig.volume
      } else {
          // Narration: use default voice
          utterance.voice = AVSpeechSynthesisVoice(language: "en-US")
          utterance.rate = UserSettings.shared.defaultVoiceRate
          utterance.pitchMultiplier = UserSettings.shared.defaultVoicePitch
          utterance.volume = UserSettings.shared.defaultVoiceVolume
      }

      synthesizer.delegate = self
      synthesizer.speak(utterance)
  }

  // AVSpeechSynthesizerDelegate
  func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, didFinish utterance: AVSpeechUtterance) {
      guard let currentIndex = currentSegmentIndex else { return }
      let nextIndex = currentIndex + 1
      if nextIndex < story.segments.count {
          currentSegmentIndex = nextIndex
          speak(segment: story.segments[nextIndex], in: story)
      } else {
          isSpeaking = false
          currentSegmentIndex = nil
      }
  }
  ```
* Implement `pause()`, `resume()`, `stop()` controls.
* Expose `@Published var progress: Double` (0.0–1.0) by dividing `currentSegmentIndex / totalSegments`.

#### 6.3. UI: Playback Controls

* On “Story Detail” screen (once `segments` exist), show a “▶ Play” button.
* While playing:

  * Show “⏸ Pause” and “⏹ Stop” buttons.
  * Display a `ProgressView(value: progress)` to visualize how far through the story the user is.
  * Optionally show “Now Speaking: \<Character.name or ‘Narration’>.”
  * Allow the user to tap on any segment in a `List` to jump directly (stop current utterance, set `currentSegmentIndex`, then `speak(...)`).

#### 6.4. Handling Interruptions & Background

* Register for audio session interruptions (`AVAudioSession.interruptionNotification`) to pause/resume gracefully.
* Handle app going to background: decide if playback should continue (potentially allow background audio mode).

---

### Phase 7: Settings, Error Handling & Deployment

#### 7.1. Settings Screen

* Under “Settings” tab, allow the user to:

  1. **Update ChatGPT API Key**

     * Show a secure `TextField` (masked) bound to `UserSettings.chatGPTApiKey`.
     * “Save” button updates Keychain.
     * Validate by making a simple “list models” or “ping” request: if 401 or no response, show error.
  2. **Default TTS Parameters**

     * Sliders / steppers for default rate, pitch, volume.
     * Any new story’s narration voice will use these values by default.
  3. **App Info**

     * Version, “About,” “License,” “Feedback.”

#### 7.2. Comprehensive Error Handling

* **Story Import Errors:**

  * File unreadable (show “Could not read the chosen file.”)
  * Not a .txt (reject at filePicker level).

* **ChatGPT API Errors:**

  * Missing API key → prompt user to go to Settings.
  * Invalid API key → show “Invalid API Key — please check.”
  * Network connectivity → “Unable to reach server. Check your internet connection.”
  * Rate limit (429) → “Rate limit exceeded. Please wait and try again.”
  * Malformed JSON → “Unexpected response from ChatGPT.”

* **TTS Errors:**

  * Invalid voice identifier → fallback to default voice + log a warning.
  * AVAudioSession errors → attempt to reactivate session or show “Audio unavailable.”

* Always present enough context in alerts so that the user knows which phase (e.g., “Character extraction failed for <story title>.”)

#### 7.3. Persistence & Data Backups

* Ensure that `StoryStorageManager` writes atomically to avoid data corruption (use `Data.write(to:options:.atomic)`).
* Provide a “Backup/Export Story Data” feature: compress all JSON files into a zip and share via `UIActivityViewController`.

#### 7.4. Testing Strategy

* **Unit Tests**:

  * `ChatGPTService` with mocked URLSession responses (valid JSON, invalid JSON, HTTP 401, 429, timeouts).
  * `StoryStorageManager` reading/writing JSON to a temporary directory.
  * `SpeechManager` delegates: simulate utterance lifecycle.

* **UI Tests**:

  * File import flow (using `.txt` stub files).
  * Indexing flow: intercept network calls with a stub server.
  * Playback UI: start, pause, stop, and jump to segments.

* **Manual QA**:

  * Test with real stories of varying lengths (short story, novel excerpt, etc.).
  * Confirm voice switching accuracy & timing.
  * Check performance on older devices (iPhone 8, iPhone 11).

#### 7.5. App Icons & Launch Screens

* Design an app icon featuring a book + speech waveform motif.
* Create a Launch Screen storyboard with a subtle “StoryReader” logo centered.

#### 7.6. App Store Preparation

* Prepare a compelling App Store description:

  * “StoryReader: Listen to any story with distinct voices for each character—powered by ChatGPT & Apple’s TTS.”
  * Screenshots showing: importing a story, reviewing characters, assigning voices, playback screen.
  * Privacy policy: explain that the user’s API key is stored securely, stories remain on device, no data is shared except via ChatGPT calls.

* Fill in metadata: keywords, categories (“Education,” “Entertainment”), support URL (if applicable).

#### 7.7. Deployment Checklist

1. Confirm all features pass acceptance criteria.
2. Archive build in Xcode, validate on App Store Connect.
3. Upload screenshots (iPhone 14, 14 Pro, etc.), fill out compliance questions.
4. Submit for App Review.
5. After approval, announce release, notify early adopters.

---

## D. Future Enhancements (beyond initial MVP)

1. **Multi-language Support**

   * Accept stories in Spanish, French, etc. →

     * Pass correct `language` field to AVSpeechUtterance (e.g., “es-ES”).
     * Prompt ChatGPT to detect language & character attributes in that language.

2. **User-Editable Story Prompt**

   * Let user refine the “system prompt” for character extraction (for specialized needs).

3. **Offline Caching**

   * Cache ChatGPT responses locally so that if user re-indexes the same story, they don’t re-pay/make API calls.

4. **Cloud Sync (iCloud)**

   * Store stories & character metadata in iCloud Drive so users can sync across devices.

5. **Custom Character Profiles**

   * Let user manually add a new character (name + attributes) if ChatGPT misses minor characters.

6. **Chapter/Bookmark Support**

   * Recognize chapters; allow bookmarks at arbitrary segments.

7. **Support for EPUB / PDF**

   * Extend file importer for EPUB (parse XHTML content) or PDF (text extraction) to broaden format support.

8. **User-Selectable Voice Presets**

   * Pre-baked “profiles” (e.g., “Dramatic Narrator,” “Cartoon Voices”) that the user can choose from.

---

### Phase Milestones Summary

1. **Phase 1 (Project Setup):**

   * Xcode project scaffolding, module separation, basic UI for API key.

2. **Phase 2 (File Import):**

   * FileImporter for `.txt`; Story persistence in local JSON; Stories list UI.

3. **Phase 3 (Character Extraction):**

   * ChatGPTService.extractCharacters; UI to “Index Story”; display character list.

4. **Phase 4 (Voice Assignment):**

   * ChatGPTService.suggestVoice (or local heuristic); bulk assign & manual override UI.

5. **Phase 5 (Story Segmentation):**

   * ChatGPTService.annotateStory or local parser; store `[StorySegment]`; annotation UI.

6. **Phase 6 (Playback):**

   * `SpeechManager` with AVSpeechSynthesizer; sequential TTS engine; playback UI.

7. **Phase 7 (Settings & Deployment):**

   * Settings screen (API key, TTS defaults); error handling; testing; App Store packaging.

---

## E. Conclusion

This plan provides a comprehensive, step-by-step roadmap to implement the StoryReader iPhone app. Each phase is broken down into concrete tasks, covering:

* **Core services** (ChatGPT integration, file I/O, TTS)
* **Data modeling** (Story, Character, VoiceConfig, StorySegment)
* **UI/UX** (SwiftUI screens for import, indexing, character review, playback)
* **Error handling**, **testing**, and **deployment** best practices.

As the sole implementer, you can now take each phase as a self-contained “ticket” or “sprint”:

1. **Phase 1 →** Set up project, modules, basic API key flow.
2. **Phase 2 →** File import + local persistence.
3. **Phase 3 →** ChatGPT character extraction.
4. **Phase 4 →** Voice selection & customization.
5. **Phase 5 →** Story segmentation & annotation.
6. **Phase 6 →** Playback engine & UI.
7. **Phase 7 →** Settings, error handling, final polish, App Store deployment.

Feel free to reference this document as you proceed. Once you confirm which phase you want to tackle first, I can begin implementing it in detail.
