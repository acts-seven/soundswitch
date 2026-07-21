# DeepMix iOS Product and Technical Design

**Status:** Approved for implementation  
**Working title:** DeepMix  
**Target:** iOS 26 minimum, capability-gated iOS 27 enhancements  
**Primary genre focus:** underground electronic music  

## Product goal

DeepMix is a high-fidelity automatic DJ-style player for playlists visible in Apple’s Music app. It preserves the effortless experience of selecting music and starting a continuous mix while adding phrase-aware sequencing, electronic-music-specific transition planning, strict-playlist intelligent shuffle, scene archetypes, configurable crossfades, and private on-device learning.

The app is legally and visually distinct from Serato Pyro. It does not reuse Serato branding, assets, copy, proprietary data, or named artist presets.

## Confirmed product decisions

- The app supports music and playlists visible through Apple Music and the local media library.
- iOS 26 is the deployment minimum.
- iOS 27 devices receive enhanced analysis through Music Understanding when the audio source is readable.
- The default experience is fully automatic.
- The first-class genre target is electronic music: minimal, micro house, house, techno, electro, breaks, ambient, and adjacent underground styles.
- Playback uses a hybrid engine.
- Intelligent Shuffle is strict: it may only use tracks in the selected playlist snapshot.
- Users choose a set intention at the start of each session.
- Users choose a global scene archetype or a private learned profile.
- Profiles may eventually learn from both ordered playlists and imported full DJ mix recordings.
- Raw audio and personal learning remain on-device by default.
- Crossfade behaviour is configurable.
- Hi-fi playback and transparent output-quality reporting are core requirements.

## User experience

### Session start

1. Choose a playlist.
2. Choose a set intention.
3. Choose a scene archetype or saved profile.
4. Review crossfade mode if desired.
5. Tap **Start Intelligent Shuffle**.

### Set intentions

- Warm Up
- Steady Groove
- Build to Peak
- Afterhours
- Deep Journey
- High Energy
- Surprise Me

Each intention defines an energy target curve, preferred transition density, peak frequency, harmonic-risk tolerance, and willingness to defer difficult tracks.

### Scene archetypes

Initial presets are geographically and musically descriptive rather than celebrity imitations:

- Romanian Minimal
- Berlin Afterhours
- Detroit Electro
- UK Breaks
- Japanese Deep House
- Melbourne Bush Techno
- Ambient Drift

A scene profile is a transparent set of planner priors, not an endorsement claim. Users can inspect and adjust values such as blend length, harmonic strictness, vocal tolerance, groove similarity, energy slope, and adventurousness.

### Now Playing

The primary playback screen shows:

- Current track and artwork
- Incoming track and artwork
- Enhanced Mix or Standard Mix capability
- Output route and audio-quality state
- Energy-arc position
- Planned transition countdown
- Concise transition rationale, such as Phrase Match, Groove Hold, Peak Release, or Safer Standard Handoff
- Play/pause, skip, pin-next, keep-transition, and reject-transition controls
- Compact access to the queue and transition settings

## Hybrid capability model

Capability is evaluated per track.

### Enhanced Mix

Eligible when the app can obtain readable audio through an asset URL or another supported PCM source.

Enhanced Mix may use:

- Beat and downbeat analysis
- Bar and phrase boundaries
- Structural sections
- BPM and tempo confidence
- Musical key over time
- Energy and pace over time
- Vocal, drum, and bass activity
- Loudness and gain matching
- Candidate intro and outro windows
- Conservative time-stretching
- Per-deck EQ and gain automation
- Dual-deck rendering through AVAudioEngine

### Standard Mix

Used when a track is playable through MusicKit but cannot lawfully or technically enter the custom PCM pipeline.

Standard Mix provides:

- Strict-playlist sequencing
- Metadata and available analysis-summary ranking
- MusicKit-native playback
- Native crossfade where available
- Optional start/end trims only after real-device validation
- Safer, shorter, lower-assumption transitions

The UI must never describe Standard Mix as beatmatched or custom-EQ blended.

## Intelligent Shuffle

The selected playlist is snapshotted when the session begins. That immutable snapshot is the only candidate pool.

The planner may:

- Reorder tracks
- Temporarily defer unsuitable tracks
- Reconsider deferred tracks later
- Respect pinned tracks
- Adapt after skips and feedback
- Select Enhanced or Standard transition strategies

The planner may not:

- Pull tracks from another playlist
- Expand into the wider library
- Start artist radio
- Silently replace an unavailable item

### Planning approach

Use a rolling constraint-aware beam search rather than a greedy next-track choice.

For each planning step:

1. Generate candidate next tracks from the unused snapshot members.
2. Generate viable transition windows for Enhanced candidates.
3. Score immediate transition compatibility.
4. Estimate future set health across the next several tracks.
5. Keep the strongest candidate paths.
6. Commit only the next transition while retaining alternatives for replanning.

### Transition score

The initial score is:

`0.22 phrase + 0.16 tempo + 0.15 groove + 0.14 energy + 0.12 window + 0.10 harmony + 0.07 vocalSafety + 0.04 loudness - penalties`

Penalties include:

- Repeated artist crowding
- Abrupt energy cliffs
- Excessive energy spikes outside the selected intention
- Vocal collisions
- Bass overlap without a planned handoff
- Harmonic conflict beyond profile tolerance
- Low-confidence beat or phrase grids
- Standard Mix limitations
- Future-path dead ends

Weights are profile-configurable and will later be personalised from feedback.

## Crossfade and transition settings

### User-facing modes

- Automatic
- Short: 4–8 bars where Enhanced, short duration where Standard
- Balanced: 8–16 bars where Enhanced
- Long: 16–32 bars where Enhanced
- Fixed Duration: seconds-based
- Off

### Advanced controls

- Preserve breakdowns
- Avoid vocal overlap
- Bass-swap intensity
- Tempo adjustment limit
- Harmonic strictness
- Prefer Enhanced Mix tracks
- Preserve full tracks

Enhanced Mix interprets a transition as cue timing, phrase alignment, gain, EQ, time-pitch, and overlap automation. Standard Mix uses the best supported MusicKit transition without pretending to provide unavailable DSP.

## Hi-fi audio requirements

- Preserve source quality wherever the selected playback API allows.
- Avoid unnecessary sample-rate conversion.
- Configure AVAudioSession for long-form playback.
- Display the active route.
- Display available quality traits such as Lossless, Hi-Res Lossless, Local, and Apple Digital Master when metadata supports them.
- Clearly explain that actual output depends on Apple Music settings, source availability, hardware route, and external DAC capability.
- Disable hidden loudness-normalisation effects in the Enhanced path; apply only explicit transition gain planning.
- Keep DSP headroom to avoid inter-sample clipping during overlaps.

## Analysis architecture

### Common interface

All analysers return a normalised `TrackAnalysis` model so the planner does not depend on OS-specific framework types.

### iOS 27 analyser

For readable assets, wrap Music Understanding behind `MusicUnderstandingAnalyser`. Request only required analysis families and cache normalised outputs.

### iOS 26 fallback analyser

Use Apple-native components:

- AVAssetReader or AVAudioFile for decoding readable audio
- Accelerate/vDSP for spectral and vector operations
- Onset-envelope and tempogram methods for BPM and beat hypotheses
- Chroma/HPCP-style features for key estimates
- Novelty and self-similarity features for structure candidates
- Loudness estimates for gain planning
- Core ML or SoundAnalysis models for vocal and instrument activity when available

The fallback targets planner-grade features rather than exact parity with iOS 27.

## Playback architecture

### Enhanced renderer graph

Each deck contains:

`AVAudioPlayerNode -> AVAudioUnitTimePitch -> AVAudioUnitEQ -> deck mixer -> main mixer`

Two decks alternate roles. A transition scheduler applies sample-timed automation from immutable `TransitionPlan` values.

### Standard renderer

Use `ApplicationMusicPlayer` or the selected MusicKit player abstraction. Apply native queue transitions and monitor state changes. Standard playback remains isolated from the custom audio engine.

### Renderer coordinator

`PlaybackCoordinator` owns one active renderer at a time and executes explicit handoff plans. The coordinator must support:

- Enhanced to Enhanced
- Standard to Standard
- Enhanced to Standard
- Standard to Enhanced

Cross-engine handoffs default to conservative fades until precise behaviour is validated on hardware.

## Persistence

Use SwiftData for:

- Playlist snapshots
- Capability probes
- Track analyses
- Cue windows
- Scene profiles
- Learned profiles
- Session plans
- Feedback events

Cache keys include persistent track identifiers, source metadata, duration, and a versioned analysis schema. The cache must invalidate when the underlying track changes or analyser versions change.

## Private learning

V1 learns through interpretable preference updates rather than full neural retraining.

Feedback events include:

- Track skipped
- Planned transition rejected
- Transition kept
- Incoming track pinned
- Session abandoned shortly after transition
- User manually changes a profile control

The profile updater adjusts bounded weights with decay and confidence thresholds.

Later versions may add:

- Playlist-derived encoders
- Imported-mix fingerprint extraction
- On-device Core ML update tasks
- Opt-in iCloud synchronisation of compact fingerprints only

Raw imported audio is never uploaded by default.

## Imported DJ mix profiles

This is post-V1 but the architecture reserves the feature.

A mix recording may be imported privately from Files. The analyser extracts:

- Transition-boundary candidates
- Mean and variance of overlap duration
- Energy curve
- Harmonic-risk behaviour
- Tempo-change behaviour
- Vocal-overlap tolerance
- Section-entry and exit preferences

When the source playlist is available, mix-to-track matching can improve cue and transition inference. Without source tracks, produce a lower-confidence behavioural fingerprint.

Public presets must not use real DJ names without explicit licensing.

## Error handling

- Permission denied: explain required Music-library access and link to system settings.
- Apple Music authorisation unavailable: preserve local-library functionality.
- Track unavailable: mark it and replan within the same playlist snapshot.
- Analysis failure: retain the track as Standard or Playback Only rather than blocking the session.
- Audio interruption: pause safely and rebuild the active renderer graph if required.
- Route change: re-evaluate format and headroom, then resume only when safe.
- Thermal pressure: reduce analysis concurrency and visual refresh frequency before compromising real-time playback.
- Low confidence: choose shorter and safer transitions.

## Privacy and legal boundaries

- No Serato branding, artwork, copy, or visual cloning.
- No public presets named after real DJs without permission.
- No extraction, conversion, or export of protected Apple Music streams.
- No upload of raw personal-library audio by default.
- No monetisation that gates access to Apple Music service playback.
- Community sharing is outside V1.

## Testing strategy

### Unit tests

- Capability classification
- Mix scoring
- Beam-search constraints
- Energy-curve adherence
- Profile weight bounds
- Cache invalidation
- Transition-plan validation

### Integration tests

- Media-library permission flows
- Playlist snapshot correctness
- MusicKit queue setup
- Enhanced renderer graph lifecycle
- Cross-engine handoffs
- Audio interruptions and route changes

### Audio regression tests

Use generated fixtures and legally distributable clips to test:

- Beat-grid stability
- Phrase-boundary tolerance
- Gain-ramp continuity
- EQ automation continuity
- No clipping during overlaps
- Sample-rate changes

### Device validation

Required on physical iPhones for:

- Long sessions
- Bluetooth and wired routes
- External DACs
- Lock-screen controls
- Background playback
- Battery and thermal behaviour
- Standard MusicKit transitions

## V1 acceptance criteria

V1 is complete when a user can:

1. Authorise Apple Music and the media library.
2. Select an existing playlist.
3. Choose an intention and scene profile.
4. Start a strict-playlist automatic set.
5. Hear Enhanced Mix transitions for readable tracks.
6. Hear Standard Mix transitions for protected tracks.
7. Configure crossfade behaviour.
8. See honest capability and output-quality indicators.
9. Skip, pin, approve, and reject transitions.
10. Resume after common interruptions without losing the session plan.

The product must never leave the playlist snapshot or claim DSP capabilities it does not actually possess.
