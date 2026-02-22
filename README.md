<p align="center">
  <img src="logofacet.png" alt="Facet" width="200"/>
</p>

<h1 align="center">FACET</h1>

<p align="center">A portable, interactive identity runtime for Android.</p>

---

## Table of Contents

- [What is Facet](#what-is-facet)
- [What Facet is Not](#what-facet-is-not)
- [Core Value Proposition](#core-value-proposition)
- [How It Works](#how-it-works)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Project Structure](#project-structure)
- [Identity Bundle (JSON Schema)](#identity-bundle-json-schema)
- [QR Code Flow](#qr-code-flow)
- [Screen Breakdown](#screen-breakdown)
- [AI Persona System](#ai-persona-system)
- [AR Session](#ar-session)
- [On-Device LLM (Cactus)](#on-device-llm-cactus)
- [Data Storage](#data-storage)
- [Profile Versioning](#profile-versioning)
- [Security](#security)
- [Visual Design System](#visual-design-system)
- [Prerequisites](#prerequisites)
- [Build and Run](#build-and-run)
- [License](#license)

---

## What is Facet

Facet is a native Android application that turns a QR code into a living, interactive identity. When you scan someone's Facet QR code, the app downloads a structured identity bundle from a hosted URL, caches it locally, and lets you interact with that person's profile in three ways:

1. **Data** -- view their structured profile information (name, title, bio, knowledge domains, FAQ).
2. **AI Persona** -- chat with an on-device LLM that is strictly bound to the person's profile data and tone.
3. **AR Avatar** -- place a 3D GLB model of the person into your physical space using ARCore.

Everything runs on-device. No cloud inference. No external API calls for AI. The LLM runs locally via the Cactus Kotlin SDK.

---

## What Facet is Not

- It is not a QR profile viewer. The QR is just a trigger. The experience happens after.
- It is not a 3D avatar gimmick. The avatar is one layer of identity rendering, not the product.
- It is not a chatbot in AR. The AI is persona-bound and constrained to profile data only.

---

## Core Value Proposition

Every person or product gets a portable identity runtime. That identity is:

- **Self-hosted** -- you control the JSON and GLB files on your own domain.
- **Structured** -- the identity bundle follows a strict schema with validation.
- **Interactive** -- beyond a static profile card, the identity can answer questions, represent itself in 3D, and maintain conversational boundaries.
- **Offline-capable** -- once scanned and cached, profiles work without internet. The LLM runs locally.

---

## How It Works

This is the end-to-end flow from scan to interaction:

```
[QR Code]
    |
    v
[App scans QR] --> extracts HTTPS URL
    |
    v
[Fetch JSON identity bundle from URL]
    |
    v
[Validate and parse JSON]
    |
    v
[Download GLB avatar file from glb_url in JSON]
    |
    v
[Store locally]
    |-- JSON  --> /files/facet/{id}/profile.json
    |-- GLB   --> /files/facet/{id}/model.glb
    |-- Metadata --> Room database (facets table)
    |
    v
[Profile is now cached and available offline]
    |
    +--> [View Profile] -- name, title, bio, knowledge, FAQ
    |
    +--> [Chat] -- on-device LLM with persona-bound system prompt
    |
    +--> [AR Session] -- place GLB avatar in physical space via ARCore
```

### What the User Gets From the QR

The QR code contains a single HTTPS URL pointing to a JSON file:

```
https://yourdomain.com/facet/prakhar.json
```

That URL serves a structured identity bundle. The QR does not embed profile data directly. It is a pointer, not a payload.

### What Happens Next

1. The app scans the QR and extracts the URL.
2. The URL is validated (HTTPS required).
3. The JSON identity bundle is fetched and parsed.
4. The GLB avatar file referenced in the JSON is downloaded.
5. Both files are saved to internal storage under `/files/facet/{profile_id}/`.
6. Profile metadata is stored in the Room database.
7. The user is taken to the Profile Overview screen.
8. From there, the user can view data, start a chat, or launch an AR session.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI Framework | Jetpack Compose |
| AR Engine | ARCore + Sceneform 1.17.1 |
| On-Device LLM | Cactus Kotlin SDK 1.4.1-beta |
| LLM Model | qwen3-0.6 |
| Database | Room (SQLite) |
| QR Scanning | ZXing Android Embedded 4.3.0 |
| Build System | Gradle (Kotlin DSL) with KSP |
| Min SDK | 24 |
| Target SDK | 34 |

React Native was explicitly avoided. Native Kotlin eliminates bridge latency and AR compatibility issues that would arise with cross-platform frameworks.

---

## System Architecture

```
com.example.drill/
|
|-- ai/                     # LLM engine and prompt construction
|   |-- CactusEngine.kt        # Singleton LLM lifecycle manager
|   |-- ModelState.kt          # Download/init state machine
|   |-- PromptSanitizer.kt     # Input sanitization before prompt injection
|   |-- SystemPromptBuilder.kt # Persona-bound system prompt generation
|
|-- data/                   # Repository implementations
|   |-- FacetRepositoryImpl.kt # Concrete repository with network + storage
|   |-- IdGenerator.kt         # Deterministic ID generation
|
|-- domain/                 # Dependency wiring
|   |-- AppContainer.kt        # Manual DI container
|   |-- FacetRepository.kt     # Repository interface
|
|-- model/                  # Domain models
|   |-- FacetProfile.kt        # Sealed class: Person | Product
|   |-- FacetType.kt           # Enum for profile type discrimination
|
|-- network/                # HTTP layer
|   |-- SecureHttp.kt          # HTTPS-only fetcher with size limits
|   |-- ByteArrayOutputStreamCapped.kt # Bounded output stream
|
|-- storage/                # Persistence
|   |-- FacetDatabase.kt       # Room database singleton
|   |-- FacetDao.kt            # Data access object (CRUD)
|   |-- FacetEntity.kt         # Room entity definition
|   |-- FacetFileStore.kt      # File system storage for JSON and GLB
|   |-- FacetTypeConverters.kt # Room type converters
|
|-- ui/                     # Presentation layer
|   |-- FacetApp.kt            # Root composable with navigation
|   |-- Screen.kt              # Sealed class navigation destinations
|   |-- DeepLinkBus.kt         # Deep link event handling
|   |
|   |-- screens/
|   |   |-- SplashScreen.kt
|   |   |-- ModelSetupScreen.kt
|   |   |-- HomeScreen.kt
|   |   |-- ScanScreen.kt
|   |   |-- ProfileOverviewScreen.kt
|   |   |-- ChatScreen.kt
|   |
|   |-- viewmodel/
|   |   |-- BaseViewModel.kt
|   |   |-- ModelSetupViewModel.kt
|   |   |-- HomeViewModel.kt
|   |   |-- ScanViewModel.kt
|   |   |-- ProfileViewModel.kt
|   |   |-- ChatViewModel.kt
|   |
|   |-- theme/
|       |-- Color.kt           # Dark spatial color palette
|       |-- Shape.kt
|       |-- Theme.kt
|       |-- Type.kt
|
|-- ArDrillActivity.kt      # AR session activity (Sceneform + Compose overlay)
|-- MainActivity.kt         # Entry point, Cactus initialization
```

---

## Project Structure

```
Facet/
|-- app/
|   |-- build.gradle.kts          # Dependencies and build config
|   |-- proguard-rules.pro
|   |-- src/
|       |-- main/
|       |   |-- AndroidManifest.xml
|       |   |-- java/com/example/drill/   # Source code (see architecture above)
|       |   |-- res/                       # Resources, layouts, themes
|       |   |-- assets/                    # Bundled assets
|       |-- androidTest/
|       |-- test/
|
|-- gradle/
|   |-- libs.versions.toml        # Centralized version catalog
|   |-- wrapper/
|
|-- build.gradle.kts              # Root build file
|-- settings.gradle.kts
|-- gradle.properties
|-- logofacet.png
```

---

## Identity Bundle (JSON Schema)

Every Facet profile is served as a JSON file from a hosted URL. The app supports two profile types: `person` and `product`.

### Person Profile

```json
{
  "id": "prakhar_001",
  "type": "person",
  "name": "Prakhar Sharma",
  "title": "AI/ML Engineer",
  "bio": "Electronics student and AI engineer building agent systems.",
  "tone": "technical, precise, minimal",
  "knowledge": [
    "AI agents",
    "LLM infrastructure",
    "PCB hardware projects",
    "Hackathons"
  ],
  "faq": [
    {
      "question": "What are you currently building?",
      "answer": "An agent control plane for deterministic AI debugging."
    }
  ],
  "glb_url": "https://yourdomain.com/assets/prakhar.glb",
  "embedding_context": "Long technical background text here...",
  "version": 1
}
```

### Product Profile

```json
{
  "id": "product_alpha",
  "type": "product",
  "name": "AlphaBoard",
  "description": "A compact development board for edge AI applications.",
  "key_features": [
    "On-board NPU",
    "WiFi 6 support",
    "USB-C power delivery"
  ],
  "glb_url": "https://yourdomain.com/assets/alphaboard.glb",
  "version": 2
}
```

### Schema Rules

| Field | Type | Required | Max Length | Notes |
|---|---|---|---|---|
| `id` | string | yes | 80 | Unique identifier |
| `type` | string | yes | 20 | `"person"` or `"product"` |
| `name` | string | yes | 80 | Display name |
| `title` | string | person only | 80 | Professional title |
| `bio` | string | person only | 1500 | Biography |
| `description` | string | product only | 1500 | Product description |
| `tone` | string | no | 1000 | Persona tone directives |
| `knowledge` | string[] | no | 12 items, 120 chars each | Domain knowledge tags |
| `key_features` | string[] | product only | 12 items, 120 chars each | Feature list |
| `faq` | object[] | no | -- | Question/answer pairs |
| `glb_url` | string | yes | 2048 | HTTPS URL to GLB avatar file |
| `embedding_context` | string | no | 1000 | Additional context for AI |
| `version` | integer | no | -- | Defaults to 1 if omitted |

The JSON should be kept under 200KB. The GLB file should be under 20MB.

---

## QR Code Flow

### What Goes in the QR

A single HTTPS URL. Nothing else.

```
https://yourdomain.com/facet/prakhar.json
```

Do not embed profile data in the QR code. The QR is a pointer to a hosted identity bundle.

### Scan Sequence

```  
1. Camera opens via ZXing scanner
2. QR detected --> URL extracted
3. URL validated (must be HTTPS)
4. JSON fetched via SecureHttp (size-bounded, timeout-protected)
5. JSON parsed and validated against schema
6. GLB file downloaded from glb_url field
7. Files saved to internal storage:
   - /files/facet/{id}/profile.json
   - /files/facet/{id}/model.glb
8. Metadata row inserted/updated in Room database
9. User navigated to Profile Overview screen
```

### Hosting Requirements

Your website must serve:

| File | Required | Purpose |
|---|---|---|
| JSON identity file | Yes | Profile data consumed by the app |
| GLB avatar file | Yes | 3D model loaded into ARCore |
| HTML preview page | No | Optional web fallback for non-app users |

---

## Screen Breakdown

### 1. Splash Screen

Shown on app launch. Checks if the on-device LLM model is already downloaded and initialized.

### 2. Model Setup Screen (first launch only)

Handles one-time download of the qwen3-0.6 model via the Cactus SDK. Shows download progress. This screen only appears when the model has not been downloaded yet. After completion, the app proceeds to the Home screen.

### 3. Home Screen

The main hub after model setup is complete.

- Top: Facet branding
- Middle: "Scan New Card" button to open the QR scanner
- Below: List of saved Facet profiles, sorted by last opened
- Each entry shows: name, type, last opened timestamp
- Tapping a profile navigates to Profile Overview

### 4. Scan Screen

Opens the camera for QR code scanning. On successful scan, processes the URL, fetches the identity bundle, and navigates to the profile.

### 5. Profile Overview Screen

Non-AR preview of a scanned profile. Displays:

- Name
- Title (for person profiles)
- Bio or description
- Action buttons:
  - **Open in AR** -- launches the AR session with the profile's GLB avatar
  - **Talk** -- opens the text chat with persona-bound AI
  - **Delete** -- removes the profile and its cached files

### 6. Chat Screen

Text-based conversation with the persona-bound AI. The on-device LLM receives a system prompt constructed from the profile data and responds within the boundaries of that identity. The LLM does not have access to information outside the profile.

### 7. AR Session (ArDrillActivity)

Full ARCore session with:

- Plane detection
- Automatic anchor placement
- GLB model rendering via Sceneform
- Compose overlay for chat input at the bottom of the screen
- Real-time AI responses while viewing the avatar in AR

---

## AI Persona System

The AI persona is the core differentiator. It is not a general-purpose chatbot. It is a constrained identity representation.

### System Prompt Construction

The `SystemPromptBuilder` constructs a system prompt from the profile JSON:

```
You are FACET, an assistant constrained to the profile below.

Rules:
- Answer ONLY using the profile information.
- If the user asks anything not covered by the profile, respond:
  "I can't answer that based on the provided profile."
- Do not use emojis.
- Keep a professional, minimal tone. Do not be friendly.

PROFILE_JSON:
{sanitized profile data}
```

### Prompt Sanitization

Before injecting profile data into the system prompt, `PromptSanitizer` applies:

- Maximum recursion depth of 4 levels
- String truncation at 800 characters
- Key count limit of 80 per object
- Array element limit of 60
- Control character stripping
- Whitespace normalization

This prevents prompt injection attacks from malicious profile JSON.

### Inference

All inference runs on-device via the Cactus SDK in `LOCAL_FIRST` mode:

```kotlin
lm.generateCompletion(
    messages = messages,
    params = CactusCompletionParams(
        mode = InferenceMode.LOCAL_FIRST,
        cactusToken = "optional_token"
    )
)
```

The LLM instance is kept loaded in memory for the duration of the app session. It is only unloaded when the app closes.

---

## AR Session

### GLB Handling

1. The GLB URL is extracted from the profile JSON (`glb_url` field).
2. The file is downloaded during the scan flow and stored locally at `/files/facet/{id}/model.glb`.
3. During an AR session, the GLB is loaded from local storage into Sceneform.
4. The model is never streamed live during an AR session.

### AR Flow

1. ARCore session starts and begins plane detection.
2. Once a horizontal plane is detected, the avatar is automatically anchored.
3. The GLB model is rendered at the anchor point with idle animation support.
4. A Compose-based chat overlay appears at the bottom of the screen.
5. The user can type messages and receive AI responses while viewing the avatar.

### Permissions

- `android.permission.CAMERA` -- required for both QR scanning and AR
- `android.hardware.camera.ar` -- required ARCore feature

---

## On-Device LLM (Cactus)

### Engine Configuration

| Parameter | Value |
|---|---|
| Model | qwen3-0.6 |
| Context Size | 2048 tokens |
| Mode | LOCAL_FIRST |
| SDK | Cactus Kotlin SDK 1.4.1-beta |

### Lifecycle

1. **First launch**: `CactusEngine` checks if the model is downloaded. If not, triggers download with progress reporting.
2. **Initialization**: After download, the model is initialized with `CactusInitParams`.
3. **Session**: The `CactusLM` instance is held as a singleton. It stays loaded in memory across screens and profile switches.
4. **Shutdown**: The model is unloaded only when the app process terminates.

### Initialization in MainActivity

```kotlin
CactusContextInitializer.initialize(this)
```

This is called in `MainActivity.onCreate()` and is mandatory per the Cactus SDK requirements.

---

## Data Storage

### Room Database

The `FacetDatabase` (SQLite via Room) stores profile metadata:

| Column | Type | Description |
|---|---|---|
| `id` | String (PK) | Profile identifier |
| `type` | FacetType | Person or Product |
| `name` | String | Display name |
| `version` | Int | Profile version number |
| `localJsonPath` | String | Path to cached JSON file |
| `localGlbPath` | String | Path to cached GLB file |
| `lastOpened` | Long | Timestamp of last access |

### File Storage

`FacetFileStore` manages the file system layout:

```
/files/facet/{profile_id}/
    profile.json    # Cached identity bundle
    model.glb       # Cached 3D avatar
```

### Multi-Profile Support

The app supports storing and switching between multiple identity bundles. Each profile is independently cached. No re-downloading occurs unless the profile version has been incremented on the server.

---

## Profile Versioning

If the identity bundle JSON contains a `version` field:

```json
{
  "version": 3
}
```

The app checks the version on profile open:

- If the remote version is newer than the cached version, the JSON and GLB are re-downloaded.
- The local cache is updated atomically.
- If the version is unchanged, the cached data is used directly.

---

## Security

### Network

- All HTTP requests enforce HTTPS. Plain HTTP URLs are rejected.
- Connection timeout: 10 seconds. Read timeout: 15 seconds.
- Maximum 3 redirects followed.

### Download Limits

- JSON response bodies are size-bounded during download (capped output stream).
- GLB files should not exceed 20MB.

### Prompt Safety

- All profile text is sanitized before injection into the LLM system prompt.
- Control characters are stripped.
- Strings are truncated to prevent context overflow.
- JSON depth, key count, and array length are all bounded.

### Data Validation

- JSON structure is validated at parse time with strict field requirements.
- Missing required fields cause parse failure.
- String fields have maximum length enforcement.
- Array fields have maximum item count enforcement.

---

## Visual Design System

The UI follows a dark spatial aesthetic. No gradients, no glow, no neon. The design is architectural.

### Color Palette

| Token | Hex | Usage |
|---|---|---|
| Background | `#0E0F11` | Primary background |
| Surface | `#15171A` | Card and surface elements |
| Divider | `#24262B` | Separator lines |
| Text Primary | `#E6E6E6` | Headings and body text |
| Text Secondary | `#A8ACB3` | Supporting text |
| Muted | `#6F737A` | Disabled or de-emphasized text |
| Accent | `#2C3440` | Subtle accent for interactive elements |

### Design Principles

- Dark-first. No light mode.
- Minimal surface contrast. Layers are distinguished by subtle value shifts, not borders.
- Typography is clean and monospaced where appropriate.
- Spacing is generous. The interface breathes.
- No decorative elements. Every pixel serves function.

---

## Prerequisites

- Android Studio Hedgehog (2023.1.1) or newer
- JDK 8+
- Android SDK 34
- An ARCore-compatible Android device (AR features require physical hardware)
- Internet connection for first-launch model download and initial QR scans

---

## Build and Run

1. Clone the repository.

2. Open the project in Android Studio:
   ```
   File -> Open -> select the Facet/ directory
   ```

3. Sync Gradle. The project uses a Kotlin DSL build with a version catalog at `gradle/libs.versions.toml`.

4. Connect an ARCore-compatible Android device or use an emulator (AR features will not work on emulator).

5. Run the app:
   ```
   ./gradlew installDebug
   ```
   Or use the Run button in Android Studio.

6. On first launch, the app will download the qwen3-0.6 LLM model (~300-600MB depending on quantization). This requires an internet connection and may take a few minutes.

7. After model setup, scan a Facet QR code to begin.

### Hosting Your Own Identity

1. Create a JSON file following the [Identity Bundle schema](#identity-bundle-json-schema).
2. Create or obtain a GLB 3D model file.
3. Host both files on an HTTPS-enabled web server.
4. Generate a QR code containing the URL to your JSON file.
5. Scan it with Facet.

---

## License

See the project repository for license information.
