# DeepMix iOS V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an iOS 26+ automatic electronic-music player that performs strict-playlist Intelligent Shuffle, uses a per-track Enhanced/Standard capability model, provides configurable phrase-aware transitions for readable local audio, and uses MusicKit-native playback for protected Apple Music items.

**Architecture:** A shared planner consumes normalised track analyses and produces immutable transition plans. Source adapters, analysers, and renderers sit behind protocols so iOS 26 fallback analysis, iOS 27 Music Understanding, AVAudioEngine Enhanced playback, and MusicKit Standard playback remain independently testable and replaceable.

**Tech Stack:** Swift 6, SwiftUI, SwiftData, MusicKit, MediaPlayer, AVFAudio, AVFoundation, Accelerate/vDSP, Core ML, SoundAnalysis, XCTest, Xcode 26.

## Global Constraints

- Minimum deployment target is iOS 26.0.
- iOS 27 Music Understanding integration must be capability-gated with `if #available(iOS 27.0, *)`.
- Intelligent Shuffle must never use a track outside the selected immutable playlist snapshot.
- The app must not use Serato names, logos, assets, copy, or pixel-identical layouts.
- Public scene profiles must not use real DJ names without a licence.
- Protected Apple Music audio must never be exported, converted, decoded into a custom PCM pipeline, or described as Enhanced Mix unless the source is actually readable.
- Apple Music playback and Standard Mix functionality must not be gated behind payment.
- Raw library audio and imported mix recordings remain on-device by default.
- Enhanced transitions must preserve DSP headroom and reject plans predicted to clip.
- Every task ends with independently executable tests and a focused commit.

---

## Planned file structure

```text
DeepMix/
├── DeepMixApp.swift
├── App/
│   ├── AppModel.swift
│   ├── CompositionRoot.swift
│   └── DeepMixEntitlements.entitlements
├── Domain/
│   ├── Track.swift
│   ├── PlaylistSnapshot.swift
│   ├── TrackCapability.swift
│   ├── TrackAnalysis.swift
│   ├── CueWindow.swift
│   ├── MixProfile.swift
│   ├── SetIntention.swift
│   ├── TransitionPlan.swift
│   └── MixSession.swift
├── Library/
│   ├── MusicAuthorisationService.swift
│   ├── MediaLibraryPlaylistSource.swift
│   ├── MusicKitPlaylistSource.swift
│   ├── PlaylistSnapshotBuilder.swift
│   └── TrackCapabilityProbe.swift
├── Analysis/
│   ├── TrackAnalyser.swift
│   ├── AnalysisCoordinator.swift
│   ├── Fallback/
│   │   ├── PCMReader.swift
│   │   ├── OnsetEnvelopeExtractor.swift
│   │   ├── TempoEstimator.swift
│   │   ├── KeyEstimator.swift
│   │   ├── StructureEstimator.swift
│   │   ├── LoudnessEstimator.swift
│   │   └── IOS26TrackAnalyser.swift
│   └── MusicUnderstanding/
│       └── IOS27MusicUnderstandingAnalyser.swift
├── Planning/
│   ├── MixScoreCalculator.swift
│   ├── EnergyCurve.swift
│   ├── CandidateGenerator.swift
│   ├── BeamSearchPlanner.swift
│   └── RollingSetPlanner.swift
├── Playback/
│   ├── PlaybackRenderer.swift
│   ├── PlaybackCoordinator.swift
│   ├── AudioSessionController.swift
│   ├── Enhanced/
│   │   ├── DeckGraph.swift
│   │   ├── TransitionAutomation.swift
│   │   └── EnhancedMixRenderer.swift
│   └── Standard/
│       └── StandardMixRenderer.swift
├── Persistence/
│   ├── AnalysisRecord.swift
│   ├── FeedbackRecord.swift
│   ├── ProfileRecord.swift
│   └── AnalysisStore.swift
├── Learning/
│   ├── FeedbackEvent.swift
│   └── ProfileUpdater.swift
├── Features/
│   ├── Library/
│   │   ├── PlaylistListView.swift
│   │   └── PlaylistListViewModel.swift
│   ├── StartMix/
│   │   ├── StartMixView.swift
│   │   └── StartMixViewModel.swift
│   ├── Player/
│   │   ├── NowPlayingView.swift
│   │   ├── NowPlayingViewModel.swift
│   │   ├── EnergyArcView.swift
│   │   └── QueueView.swift
│   └── Settings/
│       ├── TransitionSettings.swift
│       └── TransitionSettingsView.swift
└── Resources/
    ├── Assets.xcassets
    └── PrivacyInfo.xcprivacy

DeepMixTests/
├── Domain/
├── Library/
├── Analysis/
├── Planning/
├── Playback/
├── Learning/
└── Features/
```

---

### Task 1: Create the native project shell and composition root

**Files:**
- Create: `DeepMix/DeepMixApp.swift`
- Create: `DeepMix/App/AppModel.swift`
- Create: `DeepMix/App/CompositionRoot.swift`
- Create: `DeepMix/App/DeepMixEntitlements.entitlements`
- Create: `DeepMix/Resources/PrivacyInfo.xcprivacy`
- Modify: Xcode target Info settings
- Test: `DeepMixTests/App/CompositionRootTests.swift`

**Interfaces:**
- Produces: `@MainActor final class AppModel: ObservableObject`
- Produces: `struct CompositionRoot`
- Produces: `func makeAppModel() -> AppModel`

- [ ] **Step 1: Create the iOS app and test targets**

Create an iOS SwiftUI application named `DeepMix`, set Swift language mode to Swift 6, set deployment target to iOS 26.0, and add a unit-test target named `DeepMixTests`.

- [ ] **Step 2: Add required capabilities and usage descriptions**

Add Background Modes → Audio and configure:

```text
NSAppleMusicUsageDescription = DeepMix needs access to your Music library so you can choose playlists and create automatic mixes.
UIBackgroundModes = audio
```

Use the following entitlements content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict/>
</plist>
```

- [ ] **Step 3: Write the failing composition-root test**

```swift
import XCTest
@testable import DeepMix

final class CompositionRootTests: XCTestCase {
    @MainActor
    func testMakeAppModelStartsIdle() {
        let model = CompositionRoot.preview.makeAppModel()
        XCTAssertEqual(model.phase, .idle)
    }
}
```

- [ ] **Step 4: Run the test to verify it fails**

Run:

```bash
xcodebuild test -scheme DeepMix -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:DeepMixTests/CompositionRootTests
```

Expected: FAIL because `CompositionRoot` and `AppModel` do not exist.

- [ ] **Step 5: Implement the minimal app model and root**

```swift
import Foundation

@MainActor
final class AppModel: ObservableObject {
    enum Phase: Equatable {
        case idle
        case requestingAuthorisation
        case ready
        case failed(String)
    }

    @Published private(set) var phase: Phase = .idle
}
```

```swift
struct CompositionRoot {
    static let live = CompositionRoot()
    static let preview = CompositionRoot()

    @MainActor
    func makeAppModel() -> AppModel {
        AppModel()
    }
}
```

```swift
import SwiftUI

@main
struct DeepMixApp: App {
    @StateObject private var appModel = CompositionRoot.live.makeAppModel()

    var body: some Scene {
        WindowGroup {
            Text("DeepMix")
                .environmentObject(appModel)
        }
    }
}
```

- [ ] **Step 6: Run all tests**

Run the command from Step 4 without `-only-testing`.

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add DeepMix DeepMixTests
git commit -m "chore: initialise DeepMix iOS project"
```

---

### Task 2: Define stable domain models and invariants

**Files:**
- Create: all files under `DeepMix/Domain/`
- Test: `DeepMixTests/Domain/DomainModelTests.swift`

**Interfaces:**
- Produces: `Track`, `PlaylistSnapshot`, `TrackCapability`, `TrackAnalysis`, `CueWindow`, `MixProfile`, `SetIntention`, `TransitionPlan`, `MixSession`
- Consumes: Foundation only

- [ ] **Step 1: Write failing invariant tests**

```swift
import XCTest
@testable import DeepMix

final class DomainModelTests: XCTestCase {
    func testPlaylistSnapshotRejectsDuplicateEntryIDs() throws {
        let track = Track(id: "track-1", title: "3/4", artist: "Dr. Chemtrails", duration: 493.621)
        XCTAssertThrowsError(
            try PlaylistSnapshot(
                id: "playlist-1",
                title: "Test",
                entries: [
                    .init(entryID: "entry-1", track: track),
                    .init(entryID: "entry-1", track: track)
                ]
            )
        )
    }

    func testTransitionPlanRejectsTrackOutsideSnapshot() throws {
        let a = Track(id: "a", title: "A", artist: "Artist", duration: 300)
        let b = Track(id: "b", title: "B", artist: "Artist", duration: 300)
        let snapshot = try PlaylistSnapshot(
            id: "p",
            title: "P",
            entries: [.init(entryID: "ea", track: a)]
        )

        XCTAssertThrowsError(
            try TransitionPlan(
                fromTrackID: a.id,
                toTrackID: b.id,
                engineMode: .standard,
                overlapSeconds: 8,
                score: 0.8,
                rationale: .saferStandardHandoff,
                snapshot: snapshot
            )
        )
    }
}
```

- [ ] **Step 2: Run the test and verify failure**

Expected: FAIL because domain models do not exist.

- [ ] **Step 3: Implement the core models**

Use value types and `Sendable` conformance:

```swift
import Foundation

struct Track: Identifiable, Hashable, Codable, Sendable {
    let id: String
    let title: String
    let artist: String
    let duration: TimeInterval
    var albumTitle: String?
    var artworkIdentifier: String?
    var source: Source = .mediaLibrary

    enum Source: String, Codable, Sendable {
        case mediaLibrary
        case appleMusic
    }
}
```

```swift
struct PlaylistSnapshot: Identifiable, Codable, Sendable {
    struct Entry: Identifiable, Codable, Sendable {
        let entryID: String
        let track: Track
        var id: String { entryID }
    }

    enum ValidationError: Error { case duplicateEntryID }

    let id: String
    let title: String
    let entries: [Entry]

    init(id: String, title: String, entries: [Entry]) throws {
        guard Set(entries.map(\.entryID)).count == entries.count else {
            throw ValidationError.duplicateEntryID
        }
        self.id = id
        self.title = title
        self.entries = entries
    }

    func contains(trackID: String) -> Bool {
        entries.contains { $0.track.id == trackID }
    }
}
```

Define:

```swift
enum MixEngineMode: String, Codable, Sendable { case enhanced, standard, playbackOnly }
enum TransitionRationale: String, Codable, Sendable {
    case phraseMatch, grooveHold, energyBuild, peakRelease, saferStandardHandoff
}
```

`TransitionPlan` must validate both track IDs against its snapshot and enforce `overlapSeconds >= 0` and `score` in `0...1`.

- [ ] **Step 4: Implement set intentions and profiles**

```swift
enum SetIntention: String, CaseIterable, Codable, Sendable {
    case warmUp, steadyGroove, buildToPeak, afterhours, deepJourney, highEnergy, surpriseMe
}

struct MixProfile: Identifiable, Codable, Sendable {
    let id: String
    var name: String
    var phraseWeight: Double
    var tempoWeight: Double
    var grooveWeight: Double
    var energyWeight: Double
    var windowWeight: Double
    var harmonyWeight: Double
    var vocalSafetyWeight: Double
    var loudnessWeight: Double
    var preferredBlendBars: ClosedRange<Int>
}
```

Provide static presets for the seven approved scene archetypes. Ensure each weight set totals `1.0 ± 0.0001` in tests.

- [ ] **Step 5: Run tests and commit**

```bash
git add DeepMix/Domain DeepMixTests/Domain
git commit -m "feat: define mixing domain models"
```

---

### Task 3: Authorise Music access and build strict playlist snapshots

**Files:**
- Create: `DeepMix/Library/MusicAuthorisationService.swift`
- Create: `DeepMix/Library/MediaLibraryPlaylistSource.swift`
- Create: `DeepMix/Library/MusicKitPlaylistSource.swift`
- Create: `DeepMix/Library/PlaylistSnapshotBuilder.swift`
- Test: `DeepMixTests/Library/PlaylistSnapshotBuilderTests.swift`

**Interfaces:**
- Produces: `protocol PlaylistSource`
- Produces: `func playlists() async throws -> [LibraryPlaylist]`
- Produces: `func snapshot(playlistID: String) async throws -> PlaylistSnapshot`

- [ ] **Step 1: Define source DTOs and protocol**

```swift
struct LibraryPlaylist: Identifiable, Sendable {
    let id: String
    let title: String
    let trackCount: Int
}

protocol PlaylistSource: Sendable {
    func playlists() async throws -> [LibraryPlaylist]
    func tracks(in playlistID: String) async throws -> [Track]
}
```

- [ ] **Step 2: Write a failing strict-snapshot test**

```swift
actor StubPlaylistSource: PlaylistSource {
    let tracksByPlaylist: [String: [Track]]

    func playlists() async throws -> [LibraryPlaylist] { [] }
    func tracks(in playlistID: String) async throws -> [Track] {
        tracksByPlaylist[playlistID, default: []]
    }
}

final class PlaylistSnapshotBuilderTests: XCTestCase {
    func testSnapshotContainsOnlySelectedPlaylistTracks() async throws {
        let selected = [
            Track(id: "1", title: "3/4", artist: "Dr. Chemtrails", duration: 493.621),
            Track(id: "2", title: "Bass5", artist: "Dr. Chemtrails", duration: 455.750)
        ]
        let source = StubPlaylistSource(tracksByPlaylist: ["selected": selected, "other": [
            Track(id: "3", title: "Sumbody", artist: "Dr. Chemtrails", duration: 405)
        ]])
        let builder = PlaylistSnapshotBuilder(source: source)

        let snapshot = try await builder.snapshot(id: "selected", title: "Selected")

        XCTAssertEqual(snapshot.entries.map(\.track.id), ["1", "2"])
    }
}
```

- [ ] **Step 3: Implement the snapshot builder**

```swift
struct PlaylistSnapshotBuilder {
    let source: any PlaylistSource

    func snapshot(id: String, title: String) async throws -> PlaylistSnapshot {
        let tracks = try await source.tracks(in: id)
        return try PlaylistSnapshot(
            id: id,
            title: title,
            entries: tracks.enumerated().map { index, track in
                .init(entryID: "\(id):\(index):\(track.id)", track: track)
            }
        )
    }
}
```

- [ ] **Step 4: Implement Music authorisation**

Wrap `MusicAuthorization.request()` and `MPMediaLibrary.requestAuthorization` in one service. Return a domain result with separate Apple Music and media-library states so local functionality can continue when subscription authorisation is unavailable.

- [ ] **Step 5: Implement MediaPlayer playlist loading**

Use `MPMediaQuery.playlists()` and map each `MPMediaPlaylist.items` entry into `Track`. Preserve playlist order exactly. Use persistent IDs as stable IDs and include duration and artwork identifiers.

- [ ] **Step 6: Implement the MusicKit source adapter**

Fetch personal-library playlist metadata through MusicKit only where needed. Do not use this adapter to broaden a selected playlist’s candidate pool.

- [ ] **Step 7: Run tests and commit**

```bash
git add DeepMix/Library DeepMixTests/Library
git commit -m "feat: load strict Music library playlist snapshots"
```

---

### Task 4: Probe per-track Enhanced and Standard capability

**Files:**
- Create: `DeepMix/Library/TrackCapabilityProbe.swift`
- Test: `DeepMixTests/Library/TrackCapabilityProbeTests.swift`

**Interfaces:**
- Produces: `protocol CapabilityProbing`
- Produces: `func probe(_ item: MediaItemDescriptor) async -> TrackCapability`

- [ ] **Step 1: Define the descriptor and capability model**

```swift
struct MediaItemDescriptor: Sendable {
    let trackID: String
    let assetURL: URL?
    let isCloudItem: Bool
    let isProtected: Bool
    let hasMusicKitPlayableID: Bool
}

struct TrackCapability: Codable, Equatable, Sendable {
    let mode: MixEngineMode
    let readableAssetURL: URL?
    let reason: Reason

    enum Reason: String, Codable, Sendable {
        case readableLocalAsset
        case protectedAppleMusic
        case cloudPlaybackOnly
        case unavailable
    }
}
```

- [ ] **Step 2: Write the classification matrix test**

```swift
final class TrackCapabilityProbeTests: XCTestCase {
    func testReadableUnprotectedAssetIsEnhanced() async {
        let descriptor = MediaItemDescriptor(
            trackID: "1",
            assetURL: URL(fileURLWithPath: "/tmp/test.m4a"),
            isCloudItem: false,
            isProtected: false,
            hasMusicKitPlayableID: true
        )
        let result = await TrackCapabilityProbe().probe(descriptor)
        XCTAssertEqual(result.mode, .enhanced)
    }

    func testProtectedPlayableItemIsStandard() async {
        let descriptor = MediaItemDescriptor(
            trackID: "2",
            assetURL: nil,
            isCloudItem: true,
            isProtected: true,
            hasMusicKitPlayableID: true
        )
        let result = await TrackCapabilityProbe().probe(descriptor)
        XCTAssertEqual(result.mode, .standard)
    }
}
```

- [ ] **Step 3: Implement explicit decision logic**

```swift
struct TrackCapabilityProbe: CapabilityProbing {
    func probe(_ item: MediaItemDescriptor) async -> TrackCapability {
        if let url = item.assetURL, !item.isProtected {
            return .init(mode: .enhanced, readableAssetURL: url, reason: .readableLocalAsset)
        }
        if item.hasMusicKitPlayableID {
            return .init(
                mode: .standard,
                readableAssetURL: nil,
                reason: item.isProtected ? .protectedAppleMusic : .cloudPlaybackOnly
            )
        }
        return .init(mode: .playbackOnly, readableAssetURL: nil, reason: .unavailable)
    }
}
```

- [ ] **Step 4: Run tests and commit**

```bash
git add DeepMix/Library/TrackCapabilityProbe.swift DeepMixTests/Library/TrackCapabilityProbeTests.swift
git commit -m "feat: classify track mixing capability"
```

---

### Task 5: Persist versioned analyses and probe results

**Files:**
- Create: `DeepMix/Persistence/AnalysisRecord.swift`
- Create: `DeepMix/Persistence/FeedbackRecord.swift`
- Create: `DeepMix/Persistence/ProfileRecord.swift`
- Create: `DeepMix/Persistence/AnalysisStore.swift`
- Test: `DeepMixTests/Persistence/AnalysisStoreTests.swift`

**Interfaces:**
- Produces: `protocol AnalysisStoring`
- Produces: `func analysis(for key: AnalysisCacheKey) async throws -> TrackAnalysis?`
- Produces: `func save(_ analysis: TrackAnalysis, for key: AnalysisCacheKey) async throws`

- [ ] **Step 1: Define a deterministic cache key**

```swift
struct AnalysisCacheKey: Hashable, Codable, Sendable {
    let trackID: String
    let durationMilliseconds: Int
    let sourceFingerprint: String
    let analyserVersion: Int
}
```

- [ ] **Step 2: Write cache invalidation tests**

Test that a different duration, source fingerprint, or analyser version produces a cache miss while an identical key returns the saved analysis.

- [ ] **Step 3: Implement SwiftData records**

Store compact JSON-encoded normalised analyses, not Apple framework objects. Add a unique index over the serialised cache key.

- [ ] **Step 4: Implement the actor-isolated store**

```swift
actor AnalysisStore: AnalysisStoring {
    private let container: ModelContainer

    init(container: ModelContainer) {
        self.container = container
    }

    func analysis(for key: AnalysisCacheKey) async throws -> TrackAnalysis? {
        // Fetch exact key and decode TrackAnalysis.
    }

    func save(_ analysis: TrackAnalysis, for key: AnalysisCacheKey) async throws {
        // Upsert exact key and persist.
    }
}
```

The final implementation must replace the comments with concrete SwiftData fetch and upsert code before committing.

- [ ] **Step 5: Run tests and commit**

```bash
git add DeepMix/Persistence DeepMixTests/Persistence
git commit -m "feat: cache versioned track analyses"
```

---

### Task 6: Build the iOS 26 fallback analysis pipeline

**Files:**
- Create: all files under `DeepMix/Analysis/Fallback/`
- Create: `DeepMix/Analysis/TrackAnalyser.swift`
- Create: `DeepMix/Analysis/AnalysisCoordinator.swift`
- Test: `DeepMixTests/Analysis/FallbackAnalysisTests.swift`
- Add fixtures: `DeepMixTests/Fixtures/click-120bpm.wav`, `DeepMixTests/Fixtures/click-128bpm.wav`

**Interfaces:**
- Produces: `protocol TrackAnalyser`
- Produces: `func analyse(track: Track, assetURL: URL) async throws -> TrackAnalysis`
- Produces: normalised BPM, beat times, phrase candidates, key estimate, energy curve, loudness, activity summaries, and cue windows

- [ ] **Step 1: Define the analyser contract**

```swift
protocol TrackAnalyser: Sendable {
    var version: Int { get }
    func analyse(track: Track, assetURL: URL) async throws -> TrackAnalysis
}
```

- [ ] **Step 2: Write a failing deterministic tempo test**

```swift
final class FallbackAnalysisTests: XCTestCase {
    func testTempoEstimatorFinds120BPMClickTrack() async throws {
        let samples = try FixturePCM.load(named: "click-120bpm")
        let estimate = TempoEstimator().estimate(samples: samples.samples, sampleRate: samples.sampleRate)
        XCTAssertEqual(estimate.bpm, 120, accuracy: 0.5)
        XCTAssertGreaterThan(estimate.confidence, 0.8)
    }
}
```

- [ ] **Step 3: Implement PCM reading**

Use `AVAudioFile` or `AVAssetReader` to produce mono floating-point buffers at a stable analysis sample rate. Never run file decoding on the main actor.

- [ ] **Step 4: Implement onset extraction with vDSP**

Compute a short-time magnitude spectrum, positive spectral flux, adaptive thresholding, and a normalised onset envelope. Reuse FFT setup objects across frames.

- [ ] **Step 5: Implement tempo candidates**

Autocorrelate the onset envelope over a configurable electronic-music range. Resolve half-time and double-time candidates using onset periodicity and metadata priors. Return confidence and alternate hypotheses.

- [ ] **Step 6: Implement key, structure, loudness, and cue-window estimators**

- Key: chroma aggregation with major/minor template correlation.
- Structure: novelty peaks over a self-similarity representation, snapped to high-confidence bars.
- Loudness: K-weighted or documented approximation suitable for relative gain planning.
- Cue windows: score clean 8-, 16-, and 32-bar windows using vocal/activity estimates, phrase boundaries, and loudness stability.

- [ ] **Step 7: Assemble `IOS26TrackAnalyser`**

Return `TrackAnalysis` with confidence values for every inferred family. Low-confidence outputs must not be silently treated as exact.

- [ ] **Step 8: Add fixture regressions**

Assert:

- 120 BPM fixture within ±0.5 BPM
- 128 BPM fixture within ±0.5 BPM
- beat intervals are monotonic
- cue windows stay inside track duration
- energy values stay within `0...1`
- no NaN or infinite values

- [ ] **Step 9: Commit**

```bash
git add DeepMix/Analysis DeepMixTests/Analysis DeepMixTests/Fixtures
git commit -m "feat: add iOS 26 local audio analysis"
```

---

### Task 7: Add the iOS 27 Music Understanding adapter

**Files:**
- Create: `DeepMix/Analysis/MusicUnderstanding/IOS27MusicUnderstandingAnalyser.swift`
- Test: `DeepMixTests/Analysis/IOS27AnalyserMappingTests.swift`

**Interfaces:**
- Consumes: `TrackAnalyser`
- Produces: the same `TrackAnalysis` model as the fallback analyser

- [ ] **Step 1: Write mapping tests against framework-independent fixtures**

Define an internal `MusicUnderstandingResultDTO` mirroring only the fields the app uses. Test mapping of rhythm, structure, pace, key, instrument activity, and loudness into normalised domain models.

- [ ] **Step 2: Implement availability-gated adapter**

```swift
@available(iOS 27.0, *)
struct IOS27MusicUnderstandingAnalyser: TrackAnalyser {
    let version = 1

    func analyse(track: Track, assetURL: URL) async throws -> TrackAnalysis {
        let asset = AVURLAsset(
            url: assetURL,
            options: [AVURLAssetPreferPreciseDurationAndTimingKey: true]
        )
        // Create MusicUnderstandingSession, request required analyses,
        // await results, then map into TrackAnalysis.
    }
}
```

Replace the comments with concrete framework calls available in the installed iOS 27 SDK. Keep all framework-specific types in this one adapter.

- [ ] **Step 3: Select analyser by OS and capability**

`AnalysisCoordinator` must use iOS 27 analysis only when the OS is available and the track has a readable asset. Otherwise use the iOS 26 fallback.

- [ ] **Step 4: Run mapping tests and commit**

```bash
git add DeepMix/Analysis/MusicUnderstanding DeepMixTests/Analysis/IOS27AnalyserMappingTests.swift
git commit -m "feat: add iOS 27 Music Understanding analysis"
```

---

### Task 8: Implement Mix Score, energy curves, and strict candidate generation

**Files:**
- Create: `DeepMix/Planning/MixScoreCalculator.swift`
- Create: `DeepMix/Planning/EnergyCurve.swift`
- Create: `DeepMix/Planning/CandidateGenerator.swift`
- Test: `DeepMixTests/Planning/MixScoreCalculatorTests.swift`
- Test: `DeepMixTests/Planning/CandidateGeneratorTests.swift`

**Interfaces:**
- Produces: `func score(_ candidate: TransitionCandidate, context: PlanningContext) -> ScoredTransition`
- Produces: `func candidates(snapshot: PlaylistSnapshot, usedEntryIDs: Set<String>) -> [PlaylistSnapshot.Entry]`

- [ ] **Step 1: Write exact score tests**

Construct a candidate with all components equal to `1.0` and no penalties; expect `1.0`. Construct a candidate with phrase `0`, vocal collision, and a severe energy cliff; assert it ranks below a safe Standard handoff.

- [ ] **Step 2: Implement the weighted score**

```swift
struct MixScoreComponents: Sendable {
    let phrase: Double
    let tempo: Double
    let groove: Double
    let energy: Double
    let window: Double
    let harmony: Double
    let vocalSafety: Double
    let loudness: Double
    let penalties: Double
}

struct MixScoreCalculator {
    func score(_ c: MixScoreComponents, profile: MixProfile) -> Double {
        let raw = c.phrase * profile.phraseWeight
            + c.tempo * profile.tempoWeight
            + c.groove * profile.grooveWeight
            + c.energy * profile.energyWeight
            + c.window * profile.windowWeight
            + c.harmony * profile.harmonyWeight
            + c.vocalSafety * profile.vocalSafetyWeight
            + c.loudness * profile.loudnessWeight
            - c.penalties
        return min(max(raw, 0), 1)
    }
}
```

- [ ] **Step 3: Implement intention energy curves**

Each intention returns a deterministic target energy for a normalised session position `0...1`. Add tests for start, midpoint, and end values, including a peak then slight release for `buildToPeak`.

- [ ] **Step 4: Implement strict candidate generation**

Generate only unused entries from the provided snapshot. Add a test containing an attractive external track and prove it cannot appear because the generator accepts no external library source.

- [ ] **Step 5: Run tests and commit**

```bash
git add DeepMix/Planning DeepMixTests/Planning
git commit -m "feat: score strict-playlist transition candidates"
```

---

### Task 9: Implement rolling beam-search set planning

**Files:**
- Create: `DeepMix/Planning/BeamSearchPlanner.swift`
- Create: `DeepMix/Planning/RollingSetPlanner.swift`
- Test: `DeepMixTests/Planning/BeamSearchPlannerTests.swift`

**Interfaces:**
- Consumes: snapshot, analyses, capabilities, profile, intention, used entry IDs, pinned next entry
- Produces: `func nextPlan(from state: PlanningState) throws -> TransitionPlan`

- [ ] **Step 1: Write a dead-end avoidance test**

Create four tracks where the highest immediate-scoring edge leaves no viable third track, while the second-best edge enables two strong future transitions. Assert beam search chooses the second-best immediate edge.

- [ ] **Step 2: Define planning state**

```swift
struct PlanningState: Sendable {
    let snapshot: PlaylistSnapshot
    let currentEntryID: String
    let usedEntryIDs: Set<String>
    let analyses: [String: TrackAnalysis]
    let capabilities: [String: TrackCapability]
    let profile: MixProfile
    let intention: SetIntention
    let sessionPosition: Double
    let pinnedNextEntryID: String?
}
```

- [ ] **Step 3: Implement bounded beam search**

Use configurable beam width and lookahead depth, with V1 defaults of width 6 and depth 4. Score path health using immediate transition score, future transition availability, energy-curve adherence, artist spacing, and capability downgrade cost.

- [ ] **Step 4: Implement rolling replanning**

Commit only the next transition. Replan after skip, pin, unavailable track, rejected transition, or capability change.

- [ ] **Step 5: Add hard-constraint tests**

Verify:

- no external track can appear
- no used entry can repeat
- pinned next wins when playable
- unavailable pinned next produces a clear error and safe replan
- Standard items remain eligible
- low phrase confidence shortens Enhanced overlap

- [ ] **Step 6: Commit**

```bash
git add DeepMix/Planning/BeamSearchPlanner.swift DeepMix/Planning/RollingSetPlanner.swift DeepMixTests/Planning/BeamSearchPlannerTests.swift
git commit -m "feat: plan automatic sets with rolling beam search"
```

---

### Task 10: Build the Enhanced dual-deck audio renderer

**Files:**
- Create: `DeepMix/Playback/PlaybackRenderer.swift`
- Create: `DeepMix/Playback/AudioSessionController.swift`
- Create: `DeepMix/Playback/Enhanced/DeckGraph.swift`
- Create: `DeepMix/Playback/Enhanced/TransitionAutomation.swift`
- Create: `DeepMix/Playback/Enhanced/EnhancedMixRenderer.swift`
- Test: `DeepMixTests/Playback/TransitionAutomationTests.swift`
- Test: `DeepMixTests/Playback/EnhancedRendererLifecycleTests.swift`

**Interfaces:**
- Produces: `protocol PlaybackRenderer`
- Produces: `func prepare(session: MixSession) async throws`
- Produces: `func play() async throws`
- Produces: `func pause() async`
- Produces: `func execute(_ plan: TransitionPlan) async throws`

- [ ] **Step 1: Define renderer events and contract**

```swift
enum PlaybackEvent: Sendable {
    case prepared
    case started(trackID: String)
    case transitionStarted(TransitionPlan)
    case transitionCompleted(TransitionPlan)
    case paused
    case failed(String)
}

protocol PlaybackRenderer: Sendable {
    var events: AsyncStream<PlaybackEvent> { get }
    func prepare(session: MixSession) async throws
    func play() async throws
    func pause() async
    func execute(_ plan: TransitionPlan) async throws
}
```

- [ ] **Step 2: Write gain and EQ automation tests**

For a 16-bar overlap, assert:

- outgoing gain starts at 1 and ends at 0
- incoming gain starts at 0 and ends at 1
- low-frequency handoff reaches midpoint before full-range midpoint
- every curve is continuous
- combined predicted peak remains below configured headroom

- [ ] **Step 3: Build each deck graph**

Attach and connect:

```text
AVAudioPlayerNode -> AVAudioUnitTimePitch -> AVAudioUnitEQ -> AVAudioMixerNode
```

Use a minimum of three EQ bands per deck: low shelf, parametric mid, high shelf. Keep `AVAudioEngine.mainMixerNode.outputVolume` below unity headroom during overlaps.

- [ ] **Step 4: Configure the audio session**

Use `.playback`, support background playback, observe interruptions and route changes, and avoid forcing a sample rate unsupported by the active route.

- [ ] **Step 5: Execute immutable transition automation**

Schedule the incoming deck at the planned sample time, apply conservative time-pitch within the user’s tempo limit, and update ramps from a high-priority non-main execution context. Do not allocate inside the real-time render path.

- [ ] **Step 6: Add lifecycle tests with generated audio**

Use short generated PCM fixtures to verify prepare, play, pause, deck swap, transition completion, and graph rebuild after an interruption.

- [ ] **Step 7: Commit**

```bash
git add DeepMix/Playback DeepMixTests/Playback
git commit -m "feat: render Enhanced Mix dual-deck transitions"
```

---

### Task 11: Build the MusicKit Standard renderer and hybrid coordinator

**Files:**
- Create: `DeepMix/Playback/Standard/StandardMixRenderer.swift`
- Create: `DeepMix/Playback/PlaybackCoordinator.swift`
- Test: `DeepMixTests/Playback/PlaybackCoordinatorTests.swift`

**Interfaces:**
- Consumes: `PlaybackRenderer`, `TransitionPlan`, `TrackCapability`
- Produces: cross-engine handoff behaviour

- [ ] **Step 1: Write renderer-selection tests**

Assert:

- Enhanced→Enhanced uses `EnhancedMixRenderer`
- Standard→Standard uses `StandardMixRenderer`
- mixed-capability handoffs use the coordinator’s conservative bridge
- Playback Only never enters the custom engine

- [ ] **Step 2: Implement Standard playback**

Use MusicKit’s application player abstraction and native transition configuration. Configure crossfade duration from `TransitionSettings`. Keep cue trims behind a feature flag until physical-device validation proves consistent behaviour.

- [ ] **Step 3: Implement the coordinator state machine**

```swift
enum RendererState: Equatable {
    case idle
    case preparing(MixEngineMode)
    case playing(MixEngineMode, trackID: String)
    case transitioning(from: MixEngineMode, to: MixEngineMode)
    case paused
    case failed(String)
}
```

Reject illegal state transitions in unit tests.

- [ ] **Step 4: Implement conservative mixed-engine bridges**

For Enhanced↔Standard, use a validated fade-out/fade-in bridge with explicit silence-gap tolerance. Do not claim beat synchronisation for this path.

- [ ] **Step 5: Add real-device validation checklist**

Document and execute tests for:

- downloaded subscription track
- streamed subscription track
- purchased local track
- imported ALAC track
- background lock-screen playback
- wired headphones
- Bluetooth
- external DAC

- [ ] **Step 6: Commit**

```bash
git add DeepMix/Playback/Standard DeepMix/Playback/PlaybackCoordinator.swift DeepMixTests/Playback/PlaybackCoordinatorTests.swift
git commit -m "feat: coordinate Standard and Enhanced playback"
```

---

### Task 12: Implement transition settings and scene-profile controls

**Files:**
- Create: `DeepMix/Features/Settings/TransitionSettings.swift`
- Create: `DeepMix/Features/Settings/TransitionSettingsView.swift`
- Test: `DeepMixTests/Features/TransitionSettingsTests.swift`

**Interfaces:**
- Produces: `TransitionSettings`
- Consumes: planner and both renderers

- [ ] **Step 1: Define settings with safe bounds**

```swift
struct TransitionSettings: Codable, Equatable, Sendable {
    enum Mode: String, CaseIterable, Codable, Sendable {
        case automatic, short, balanced, long, fixedDuration, off
    }

    var mode: Mode = .automatic
    var fixedDurationSeconds: Double = 8
    var preserveBreakdowns = true
    var avoidVocalOverlap = true
    var bassSwapIntensity: Double = 0.65
    var tempoAdjustmentLimitPercent: Double = 4
    var harmonicStrictness: Double = 0.55
    var preferEnhancedTracks = false
    var preserveFullTracks = false

    mutating func clamp() {
        fixedDurationSeconds = min(max(fixedDurationSeconds, 0), 60)
        bassSwapIntensity = min(max(bassSwapIntensity, 0), 1)
        tempoAdjustmentLimitPercent = min(max(tempoAdjustmentLimitPercent, 0), 8)
        harmonicStrictness = min(max(harmonicStrictness, 0), 1)
    }
}
```

- [ ] **Step 2: Test bounds and planner mapping**

Assert each mode maps to the correct Enhanced bar range and Standard seconds range. Assert `off` produces zero overlap.

- [ ] **Step 3: Build the SwiftUI settings screen**

Use plain-language labels, show bar-based values for Enhanced and seconds-based behaviour for Standard, and include a concise explanation of the distinction.

- [ ] **Step 4: Persist settings with AppStorage or SwiftData**

Use one versioned settings document and migrate older versions explicitly.

- [ ] **Step 5: Commit**

```bash
git add DeepMix/Features/Settings DeepMixTests/Features/TransitionSettingsTests.swift
git commit -m "feat: add configurable transition controls"
```

---

### Task 13: Build playlist, start-mix, player, queue, and quality UI

**Files:**
- Create: all files under `DeepMix/Features/Library/`
- Create: all files under `DeepMix/Features/StartMix/`
- Create: all files under `DeepMix/Features/Player/`
- Test: `DeepMixTests/Features/StartMixViewModelTests.swift`
- Test: `DeepMixTests/Features/NowPlayingViewModelTests.swift`

**Interfaces:**
- Consumes: authorisation, playlist sources, snapshot builder, planner, playback coordinator, settings, feedback recorder
- Produces: approved end-to-end flow

- [ ] **Step 1: Implement the playlist screen state machine**

States: loading, permissionRequired, loaded, empty, failed. Display only playlists returned by authorised sources.

- [ ] **Step 2: Implement the start-mix flow**

The view model requires a selected playlist, intention, and profile before enabling Start Intelligent Shuffle. It must create a snapshot before navigating to Now Playing.

- [ ] **Step 3: Write start-flow tests**

Assert the start button is disabled until all selections exist, and that the exact selected playlist ID is passed to `PlaylistSnapshotBuilder`.

- [ ] **Step 4: Implement Now Playing**

Show:

- current and incoming artwork
- capability badge
- route and quality summary
- energy arc
- transition countdown and rationale
- play/pause, skip, pin, keep, reject

Use system materials and an original visual language; do not replicate Pyro screen geometry.

- [ ] **Step 5: Implement EnergyArcView**

Render target and observed energy curves with Swift Charts or Canvas. Mark current position and next planned peak.

- [ ] **Step 6: Implement queue visibility without manual burden**

Display the rolling plan, deferred tracks, pinned next, and Standard/Enhanced status. Allow removing or pinning only; automatic mode remains the primary interaction.

- [ ] **Step 7: Add accessibility and dynamic-type tests**

Verify identifiers, labels, reduced motion, large text, and VoiceOver reading order.

- [ ] **Step 8: Commit**

```bash
git add DeepMix/Features DeepMixTests/Features
git commit -m "feat: add automatic mix user experience"
```

---

### Task 14: Add private feedback learning

**Files:**
- Create: `DeepMix/Learning/FeedbackEvent.swift`
- Create: `DeepMix/Learning/ProfileUpdater.swift`
- Test: `DeepMixTests/Learning/ProfileUpdaterTests.swift`

**Interfaces:**
- Produces: `func applying(_ event: FeedbackEvent, to profile: MixProfile) -> MixProfile`
- Consumes: bounded interpretable profile weights

- [ ] **Step 1: Define feedback events**

```swift
enum FeedbackEvent: Codable, Sendable {
    case skipped(trackID: String, secondsPlayed: Double)
    case transitionRejected(from: String, to: String)
    case transitionKept(from: String, to: String)
    case pinned(trackID: String)
    case sessionAbandoned(afterTransitionTo: String, elapsed: Double)
}
```

- [ ] **Step 2: Write bounded-update tests**

Assert repeated events cannot drive any weight below 0.02 or above 0.50, total weights are renormalised to 1.0, and one event produces only a small update.

- [ ] **Step 3: Implement conservative updates**

Use explicit learning rates, confidence accumulation, decay, and a minimum sample threshold before visible profile changes. Store only derived events and weights.

- [ ] **Step 4: Integrate feedback controls**

Keep, reject, skip, and pin actions must record feedback asynchronously without blocking playback.

- [ ] **Step 5: Commit**

```bash
git add DeepMix/Learning DeepMixTests/Learning
git commit -m "feat: learn private mix preferences from feedback"
```

---

### Task 15: Hardening, telemetry, privacy, and release verification

**Files:**
- Modify: `DeepMix/Resources/PrivacyInfo.xcprivacy`
- Create: `DeepMix/App/Diagnostics.swift`
- Create: `docs/testing/device-matrix.md`
- Create: `docs/legal/apple-music-boundaries.md`
- Test: full test suite and physical-device matrix

**Interfaces:**
- Produces: release evidence for TestFlight

- [ ] **Step 1: Add privacy manifest declarations**

Declare only APIs actually used. Do not add third-party tracking SDKs in V1.

- [ ] **Step 2: Add local diagnostics**

Use `Logger` categories for library, analysis, planning, enhanced playback, standard playback, and learning. Never log titles, raw audio paths, or personal playlist names in production builds.

- [ ] **Step 3: Add performance signposts**

Measure analysis duration, planner duration, audio underruns, renderer handoff duration, memory pressure, and thermal-state changes.

- [ ] **Step 4: Run the complete automated suite**

```bash
xcodebuild test -scheme DeepMix -destination 'platform=iOS Simulator,name=iPhone 17 Pro'
```

Expected: all tests PASS.

- [ ] **Step 5: Run static analysis and warnings-as-errors build**

```bash
xcodebuild build -scheme DeepMix -destination 'generic/platform=iOS' SWIFT_TREAT_WARNINGS_AS_ERRORS=YES
```

Expected: BUILD SUCCEEDED with zero warnings.

- [ ] **Step 6: Complete physical-device validation**

Record results for iOS 26 and iOS 27 devices across local AAC, local ALAC, purchased music, downloaded subscription music, streamed subscription music, Bluetooth, wired output, external DAC, interruptions, incoming calls, lock screen, background playback, and a two-hour thermal test.

- [ ] **Step 7: Verify product promises**

Manually prove:

- the planner never leaves the selected snapshot
- Standard tracks never display Enhanced claims
- protected streams cannot be exported
- scene presets contain no unlicensed DJ names
- Apple Music playback remains available without payment gating
- raw audio is not uploaded

- [ ] **Step 8: Commit**

```bash
git add DeepMix/Resources DeepMix/App/Diagnostics.swift docs
git commit -m "chore: harden DeepMix for TestFlight"
```

---

## Plan self-review

### Spec coverage

- iOS 26 baseline: Tasks 1, 6, 10–15.
- iOS 27 Music Understanding enhancement: Task 7.
- Strict playlist behaviour: Tasks 3, 8, 9, and release verification.
- Enhanced/Standard hybrid: Tasks 4, 10, and 11.
- Full automatic flow: Tasks 9 and 13.
- Electronic scene archetypes and set intentions: Tasks 2, 8, 12, and 13.
- Crossfade controls: Task 12.
- Hi-fi and output route handling: Tasks 10, 11, 13, and 15.
- On-device private learning: Task 14.
- Imported full-mix profile learning: intentionally deferred from V1, with domain extension points preserved in the approved design.
- Legal and privacy boundaries: Tasks 1 and 15.

### Type consistency

- `TrackAnalyser` always returns `TrackAnalysis`.
- `TrackCapability.mode` uses `MixEngineMode`.
- `TransitionPlan` is the sole immutable renderer instruction.
- `PlaybackCoordinator` is the only component selecting or bridging renderers.
- `PlaylistSnapshot` is the only candidate-pool source accepted by planners.

### Execution order

Tasks are sequential because later interfaces depend on earlier domain contracts. Tasks 6 and 7 may be developed in parallel after Tasks 2, 4, and 5. UI Task 13 may begin with stubs after Tasks 2 and 3, but final integration depends on Tasks 9–12.
