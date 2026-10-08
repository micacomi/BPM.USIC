BPMusic

Heart-rate adaptive music player. A Wear OS watch streams heart rate to an Android phone, the phone maps that heart rate to a musical tempo, and your own locally stored music is time-stretched and swapped to match it.

Nothing is uploaded. No track is redistributed. All tempo changes happen at playback time on files you already own, which is why the copyright question stays simple.

Fourth year project, Singidunum University, Niš.

Status
Piece	Status
Tempo controller (filter, Karvonen, deadband, slew, bar-quantised commits)	done, unit tested
Safety layer (PAR-Q screening, ceiling, cooldown, hard stop)	done, unit tested
BPM detection of local files (spectral flux + autocorrelation)	done, unit tested
Track selection by BPM with half/double-time handling	done, unit tested
Playback with pitch-preserving time stretch and crossfades	done
Wear OS heart rate streaming (ExerciseClient, survives screen off)	done, works on hardware
BLE heart rate strap source (0x180D)	done
Onboarding, baseline, home, library, session UI	done
File picker import with persistable URI permissions	done
Spotify account connection and library import	done
Spotify tempo lookup	done, partial coverage
Spotify playback backend	in progress
Session summary with calories and HR zones	not started
Huawei support	not started, see below
Generated / procedural music	not started, see below
Build
Open the project root in Android Studio (Ladybug or newer).
Let it sync. If a dependency version in gradle/libs.versions.toml is stale, accept Studio's suggested bump.
:mobile is the phone app, :wear is the watch app. Both use the same applicationId on purpose; Wear OS pairs the two apps by package name.
./gradlew :core:test runs everything worth testing. :core is pure Kotlin with no Android dependencies.
Architecture
  watch                          phone
  ------                         -----
  ExerciseClient (1 Hz)
        |
        |  MessageClient  /bpmusic/hr
        v
                              WearMessageService
                                    |
                                    v
                              HeartRateSource  <- also BLE strap
                                    |
                              HeartRateFilter      median-5 then EMA
                                    |
                              TempoController      Karvonen -> safety -> BPM
                                    |                deadband -> slew limit
                                    v
                              TrackSelector        nearest BPM, least stretch
                                    |
                              PlaybackEngine       ExoPlayer x2, Sonic stretch
The parts worth understanding

Why ExerciseClient and not MeasureClient. MeasureClient is the simpler API and it is what most samples use, but the system throttles it the moment the watch screen turns off. Nobody runs with their wrist raised, so heart rate would die seconds into every session. An active exercise is what the platform guarantees to keep sampling in the background. The cost is that only one app can own an exercise at a time.

Why the slew limiter exists. A heart rate of 100 then 80 then 70 then 120 would otherwise produce four jump cuts. The controller emits a target and the slew limiter walks the actual tempo toward it at a couple of BPM per second, so you get a build-up instead of a switch.

Why tempo changes only land on bar boundaries. Changing playback rate mid-bar is audible as a wobble and instantly breaks the illusion. Note that shouldCommit counts bar indices rather than testing whether the position is within a short window of a bar line. The window version sounds equivalent and is not: the control loop ticks every 200ms, so at most tempos the window was simply stepped over and changes landed late or never.

Why we swap tracks instead of stretching further. ExoPlayer's Sonic-based stretch preserves pitch cleanly within roughly 0.85x to 1.20x and sounds terrible outside it. So TrackSelector finds a track already near the target and stretches only the residual. That is the entire reason the library is indexed by BPM.

Why the safety ceiling pushes back instead of just clamping. Faster music makes people work harder, working harder raises heart rate, higher heart rate would make the music faster. That is a positive feedback loop with a person inside it. Above the ceiling the controller actively drives the tempo down. This is the part of the design that matters most.

Why sleep mode ignores your heart rate. Every other mode follows. Sleep leads: it starts near your current rate and descends at a fixed slow rate so your heart follows the music rather than the other way around. LEAD modes deliberately bypass the deadband. An earlier version did not, and the descent (1 BPM per 30 seconds) was smaller than the deadband, so sleep mode silently froze at the starting tempo forever. There is a test for that now.

Why the onset envelope gets smoothed before autocorrelation. Without it the estimator is at the mercy of bin quantisation. A true beat period of 34.45 envelope frames correlates poorly at integer lag 34 but almost perfectly at lag 69, where the rounding error happens to be smaller, so the detector confidently reports half tempo. 128, 150 and 174 BPM all failed this way before the fix.

Why the subharmonic penalty, not harmonic summing. The intuitive fix for octave errors is to reward a candidate whose 2x and 3x lags also correlate. That is backwards: every multiple of a slow lag is also a multiple of the fast one, so it rewards slow candidates. The estimator instead penalises candidates whose half and third lags correlate, because a genuine fundamental has nothing there.

Streaming services

Spotify is supported in a reduced form, and the reasons are worth stating because they shaped the architecture.

The Spotify Android SDK exposes play, pause, skip and metadata, but no playback speed control. Time-stretching a Spotify stream is not possible, so the core mechanic of this app cannot run on one. Separately, Spotify closed its audio-features endpoint to new applications in late 2024, which is where track tempo used to come from. New API credentials receive HTTP 403 on it.

So the Spotify mode selects rather than stretches: it picks the track in your library whose tempo is nearest your target and plays it unmodified. Tempo comes from an external database matched on title and artist, which means partial coverage and occasional mismatches between a live version and a studio cut.

YouTube Music has no public API and its terms do not permit this use.

This is the clearest argument for why the full product has to run on local files.

Huawei

Huawei watches do not run Wear OS, so :wear will not install on them.

Wear Engine (recommended first). A phone-side HMS kit that exposes heart rate from a paired Huawei wearable. You write no watch code at all, just another HeartRateSource implementation. Needs a Huawei developer account and a permission application.
Native HarmonyOS app in DevEco Studio, written in ArkTS. Entirely separate codebase that only has to read the sensor and send samples. Keep Protocol.kt as the contract.

Until then, a BLE chest strap covers every Huawei user and everyone else.

Generated music

Not implemented, deliberately. Real-time neural generation on a phone is not viable yet for continuous playback: the models are too slow and the battery cost is unacceptable. Two realistic routes:

Procedural: MIDI patterns plus a sampler, rendered through Oboe. You own the output, no copyright question, and tempo is just a variable, so there is no time-stretching artifact at all.
Stem layering: pre-rendered royalty-free loops at a few base tempos, layered in and out by intensity. Cheap on CPU, sounds good.
Legal and privacy

This app is framed as entertainment and fitness. It makes no claim to diagnose, monitor or treat anything: under EU MDR that would reclassify it as a medical device and the compliance burden changes completely.

Heart rate and the health questionnaire are special-category personal data under GDPR and the Serbian ZZPL. They are stored in app-private storage and never transmitted. ProfileStore.wipe() is the right-to-erasure implementation.

Known gaps
Genre tags are not read from file metadata yet, so genre routing falls back to mode defaults.
Crossfades are volume-matched but not beat-matched. Track.beatOffsetSec is already computed for this, it just is not used yet.
Analysis runs in a plain coroutine. Move it to WorkManager with a requiresCharging constraint before shipping.
No BLE device picker UI; BleHeartRateSource takes a MAC address directly.
Sleep mode needs a much longer fade-out and an auto-stop timer.
Credits

Tempo data for Spotify tracks provided by GetSongBPM.
