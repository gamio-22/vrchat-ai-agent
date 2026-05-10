# vrchat-ai-agent

Videos available on the channel:  
https://www.youtube.com/channel/UCXj9c0VsTbFLUMKkNMEHiDg

# VRChat AI Agent — Technical Overview

> This repository does not include source code, character prompts, or personal memory data.  
> This document describes the technologies used, capabilities, and design philosophy, intended to accompany the demo videos.

## Overview

An AI agent running inside VRChat that combines voice conversation, visual recognition, long-term memory, an emotion/drive model, autonomous behavior, and OSC-based body control.

It listens to the user's voice, perceives the surroundings through a camera, generates responses using a local LLM, and replies via VOICEVOX-based TTS and the VRChat Chatbox. It also controls avatar movement, gaze, facial expressions, emotes, Use/Grab actions, and piano performance through VRChat OSC and keyboard input.

This is not a simple chatbot. The agent maintains rich internal states including: who it is talking to, people and objects in its field of view, whether it is being petted, current emotions, boredom, fatigue, loneliness, curiosity, defiance, past memories, and its relationship with each person. Based on these states, it selects actions such as speaking, looking, walking, stopping, searching, reacting, waiting, playing, and performing piano.

## Technologies Used

- Python
- VRChat OSC / python-osc
- Local LLM / Ollama-compatible models via OpenAI-compatible API
- Faster Whisper for speech recognition
- VOICEVOX-based TTS for voice output
- SpeechBrain ECAPA for speaker identification
- OpenCV for camera processing
- Ultralytics YOLO / YOLOv8 Pose for person, pose, and object detection
- Custom YOLO model for VRChat-specific object detection
- Depth Anything V2 Small for monocular depth estimation
- Qwen2.5-VL-based VLM for image observation and mirror detection
- ChromaDB + Sentence Transformers for long-term memory and search
- Working Memory for short-term conversation context management
- Voicemeeter / virtual audio device integration
- PyDirectInput for keyboard input to the VRChat window
- MIDI parsing and `.vrcpiano` / `.vrcpiano.json` score format for in-avatar piano performance
- OSC send logs and VRChat OSC response logs for performance accuracy evaluation
- Correction DB / learned scores for iterative piano performance improvement

## Features

### 1. Voice Conversation

- Captures user voice from microphone or virtual audio input
- Transcribes Japanese speech using Faster Whisper large-v3
- Filters out transcriptions that are too short, too long, or resemble prompt hallucinations
- Uses real-time VAD to detect whether the user is still speaking
- Waits for the user to finish before responding; handles interruption and stop commands
- Distinguishes between strangers, monologues, commands, questions, and emotional speech to adjust response priority
- Streams LLM responses progressively to the Chatbox
- Limits text length for TTS and queues audio playback
- Supports a lightweight mode that responds to short remarks using only the Chatbox and gaze cues

### 2. Speaker Identification

- Matches speech against a registered master voice profile
- Distinguishes between master, stranger, and unknown speakers
- Also references the VRChat OSC `IsMasterTalking` flag
- Decides whether to respond immediately or defer based on the content of stranger speech
- Tracks multiple strangers in the short term; entries expire after a set time
- Adjusts proximity, caution level, and response priority based on the identified speaker

### 3. Avatar Control

- Chatbox messages
- Forward/backward and left/right movement, stopping, and turning
- Horizontal and vertical gaze control
- Jump and repeated jump
- Sit, lie down, stand up
- Wave, clap, point, cheer
- Dance, backflip, fall-down reactions
- Use
- Grab
- Grab Hold
- Drop / Release
- Facial expression switching:
  - neutral, calm, blush, smug, sad, puzzled, smile, happy, joy, angry, fear, surprised, troubled, tongue, wink
- Show/hide in-avatar piano, sit at piano, control key parameters

### 4. Understanding and Acting on User Instructions

Detects the following intents from voice input using a rule-based system, and links LLM responses to physical actions:

- "Sit down" → sit
- "Lie down" → lie down
- "Stand up" → stand
- "Follow me" → follow the user
- "Stop" → halt movement
- "Turn right/left" → rotate
- "Wave" → wave
- "Dance" → dance
- "Look at this" → visual recognition
- "Use / Press / Open" → Use
- "Hold / Grab" → Grab
- "Keep holding" → Grab Hold
- "Let go" → Release
- "Show piano / Put away piano / Play piano" → show, hide, or perform piano

### 5. Visual Perception

- Captures VRChat screen / surroundings via camera input
- Detects people using YOLOv8 Pose
- Supports detection of multiple people simultaneously
- Estimates person position, size, and center within the frame
- Roughly estimates distance to the user
- Detects when a person enters or leaves the field of view
- Retrieves keypoints such as shoulders, hips, and wrists
- Detects hand waves, crouching, and sudden movements
- Uses VLM to describe the image and respond to requests like "Look at this"
- Suppresses false positives: if the VLM reports a person but YOLO does not detect one, the VLM result is discounted

### 6. Custom YOLO Object Detection

- Detects specific VRChat objects using a custom-trained YOLO model
- Currently active targets:
  - `door_knob`
  - `poteto_chips`
- When a `door_knob` is detected, aligns toward it and performs Jump + Use if needed
- Responds to grabbable/touchable objects like `poteto_chips` with Grab/Use actions
- Takes detection confidence, screen position, and cooldown into account
- Limits repeated reactions to the same object
- Separates person-tracking targets from object-tracking targets to avoid interference

### 7. Depth-Assisted Navigation

- Estimates relative depth from a monocular camera using Depth Anything V2 Small
- Divides the frame into left/center/right and floor/mid zones for analysis
- Determines whether the path ahead is clear or obstructed
- Estimates which side has more open space
- Controls wall-following movement, dead-end avoidance, and turn bias via DepthNavigator
- Falls back to normal visual info and conservative movement when depth data is stale or unavailable
- Supports debug overlay display for depth visualization

### 8. User Following

- Tracks a visible person as a follow target
- Stops when too close; moves forward when too far
- Adjusts heading based on how far the target is from the screen center
- Searches in the last known direction when the target is lost
- Transitions to search behavior after a prolonged loss
- Separately manages gaze, body orientation, and distance during following
- Uses appearance information captured at follow-start to reduce target mix-ups

### 9. Autonomous Behavior

Even when there is no conversation or commands, the agent acts based on its internal state:

- Looks around
- Glances around restlessly
- Wanders / explores
- Approaches objects
- Observes things it finds interesting
- Talks to itself when bored
- Sits or rests when tired
- May lie down during low-arousal periods or at night
- Stops and reacts when a user approaches
- Transitions to search behavior when a person disappears from view
- Spontaneously plays piano based on mood and boredom level
- Suppresses autonomous behavior in situations where interruption is inappropriate (e.g., mid-conversation or mid-movement)

### 10. Emotions, Drives, and Personality

Maintains internal state across two layers — emotions and drives.

Emotion examples:
- valence, arousal, happiness, affection, anger, fear, sadness, disgust, guilt, schadenfreude

Drive examples:
- fatigue, loneliness, touch_hunger, boredom, curiosity, defiance

These values affect response tone, facial expressions, movement, rest behavior, autonomous actions, and compliance with commands. For example: repeated commands increase defiance; being petted reduces loneliness and touch hunger; high boredom increases self-talk and piano playing.

### 11. Reacting to Head Pats

- Detects HeadPat from VRChat OSC parameters
- Changes facial expressions and emotional state while being petted
- Affects affection, happiness, touch_hunger, loneliness, and related values
- Includes a cooldown after pose changes to avoid false positives immediately after
- Head pat reactions take priority over conversation, movement, and autonomous behavior in certain situations

### 12. Long-Term Memory and Short-Term Context

- Saves conversation content as episodic memory
- Saves user preferences and relationship history as factual memory
- Maintains self-memory about the AI itself
- Vector search powered by ChromaDB
- Multilingual embeddings using Sentence Transformers `intfloat/multilingual-e5-small`
- Evaluates whether memories are similar, contradictory, or supplementary
- Assigns confidence and importance scores to each memory
- Recall score calculated as: confidence × decay × similarity
- Old memories decay according to their importance
- Long conversation histories are summarized and compressed
- Working Memory retains recent conversation, pending context, and emotional flow
- Recalled memories can influence the agent's current mood

### 13. Relationship Tracking

- Adjusts impression based on who the person is (master, stranger, etc.)
- Relationship changes are triggered by events: conversation, petting, being ignored, approaching, commands, etc.
- Maintains mid-term states such as trust, affinity, and wariness
- Reflects closeness, caution, and social distance in responses and behavior
- Stores per-person impressions in long-term memory

### 14. Learning Rules

The agent can retain user-given behavioral rules in the form "In this situation, do this."

Examples:
- Wave when a person is visible
- Show a specific expression when being petted
- Perform a specific action in front of a mirror
- Use interactable objects when found
- Grab and hold holdable objects when found

Rules support confirmation, addition, cancellation, and forgetting. Triggers are normalized to states such as `target_visible`, `is_headpat`, `mirror_active`, `is_autonomy`, `interactable`, or always-on, and are matched against current conditions. When defiance is high, the agent may occasionally break its own rules.

### 15. Natural Gaze Control

- Makes natural eye contact when a person is visible
- Looks at the person being spoken to during conversation
- Adds randomized gaze breaks to avoid constant staring
- Includes micro-saccade-style small eye movements
- Gaze behavior varies with interest and boredom levels
- Looks back toward the direction where a target was lost
- Prevents conflicts between object gaze and person gaze

### 16. Reacting to Mirrors and Self-Image

- When near a mirror, enters a search behavior to find the mirror or its own reflection
- Approaches avatar-like figures detected by YOLO
- Uses Qwen2.5-VL to determine whether what it sees is itself or someone else
- Responds differently with facial expressions and behavior depending on the judgment
- Suppresses other movement during mirror behavior
- Falls back to a simplified search mode if DepthNavigator is unavailable

### 17. Reacting to Objects and Poses

- Recognizes general objects grouped into categories:
  - Food, Animals, Furniture, Electronics, Other holdable/usable items
- People are not treated as objects; they are handled separately through visual recognition, following, and pose response
- Responds to objects by looking, approaching, or Using/Grabbing based on interest level
- Applies a cooldown to repeated reactions to the same object
- ROI check near the user's wrist treats nearby objects as "something the user is showing"
- Simple reactions to user hand waves, crouching, and sudden movements

### 18. Piano Performance

- Controls the show/hide of the in-avatar piano
- Operates Bool keys via OSC parameters `note_0` through `note_87`
- Detects user instructions like "Play something" or "Play this song"
- Scans `.mid` / `.midi` files in `piano_songs/` as performance candidates
- Prefers `.learned.vrcpiano.json`, `.vrcpiano`, `.vrcpiano.json`, or `.nosustain.vrcpiano.json` if available for a MIDI file
- Generates an avatar-compatible `.vrcpiano.json` from MIDI if no existing score is found
- Analyzes MIDI Note On/Off, tempo, sustain, and overlapping notes, and converts them to VRChat avatar OSC events
- Adjusts minimum press duration, re-trigger behavior, event intervals, maximum simultaneous notes, and pseudo-sustain for Bool keys
- Saves reference events, OSC send logs, and VRChat OSC response logs for each performance
- Evaluates VRChat OSC echo as `avatar_response_events` to measure missed notes, extra reactions, timing errors, and tempo drift
- When external MIDI input is enabled, prioritizes it over OSC echo for evaluation
- Stores evaluation results in a correction DB; when enabled via environment variable, auto-corrects tempo multiplier and Note On delay after sufficient observations
- Limits correction range and per-update step size to prevent sudden degradation
- Learned score auto-export is enabled via environment variable; adjusts conversion settings based on patterns like missing notes or dense bursts
- Learned scores support one-generation rollback to suppress updates in degrading directions
- Interrupted performances are treated as cancelled and excluded from strong corrections or degradation judgments
- During performance, suppresses movement, gaze, pose, Use/Grab, and autonomous reactions to prevent conflicts
- Stops immediately if the user speaks during autonomous performance
- Also stops immediately on short stop commands like "Stop" or "Quit" during manual performance
- Generates random improvisational phrases based on current mood
- Saves composed improv phrases to `piano_compositions.json` for later replay
- Optionally records external MIDI input, though VRChat does not natively output real audio as MIDI; treated as external verification only

### 19. Conversation Quality Stabilization

- Response length limiting
- Text volume limiting for TTS
- Formatting for Chatbox display
- Garbled text detection
- Repeated response detection
- Recovery responses for conversation loops
- Strips internal tags, emotion labels, piano control tags, and tool directives from Chatbox/TTS output
- Dispatches LLM streaming output to Chatbox, TTS, and OSC actions simultaneously
- Suppresses autonomous speech from calling out to someone when no one is visible

## What the Videos Demonstrate

1. Starting up from an idle state inside VRChat
2. Responding via Chatbox and voice when spoken to
3. Responding to voice commands: "Sit down," "Stand up," "Wave," "Dance," etc.
4. Following the user on command: "Follow me"
5. Searching when the user disappears from view
6. Changing expressions and emotions when petted
7. Describing visible people or objects on request: "Look at this"
8. Detecting a door knob, aligning toward it, and using it
9. Grabbing/Using objects like potato chips
10. Autonomously looking around and walking
11. Displaying the piano and performing MIDI-based or improvised scores via OSC
12. Playing the same piece multiple times and observing timing changes from log evaluation and correction
13. Reflecting remembered content in later conversation
14. Changing behavior when multiple people or strangers are present

## Highlights

- Behaves not just as a chatbot, but as an AI with a body inside VRChat
- Simultaneously handles voice, vision, memory, emotion, drives, and physical action
- Local LLM-centered design allows fine-grained control over character and response behavior
- Integrates VRChat movement, expressions, emotes, Chatbox, and avatar parameters via OSC
- Uses depth estimation to navigate around obstacles rather than simply moving forward
- Speaker identification allows separate handling of the master and other users
- Long-term memory, short-term context, and relationship tracking enable ongoing relationships
- Autonomous behavior keeps the agent present and active even when left alone
- Converts MIDI and avatar-compatible scores into VRChat avatar OSC events for in-avatar piano performance
- Compares performance logs with VRChat OSC responses to record data for timing correction and score adjustment
- Combines custom YOLO and VLM to react to world-specific objects within VRChat

## Limitations

- Currently a research / personal development prototype
- Depends on VRChat, OSC configuration, avatar-side parameters, and virtual audio environment
- Camera/screen recognition accuracy is affected by world, lighting, avatar shape, and field of view
- YOLO/VLM/depth estimation can produce false positives; suppressed via cooldown and confidence thresholds before acting
- Depth estimation is monocular and used for relative obstacle judgment, not actual distance
- Autonomous movement speed and frequency are limited for safety
- Speaker identification is affected by noise, microphone environment, and overlapping voices
- MIDI performance is affected by the avatar's Bool key specification, OSC load, and VRChat sync state
- Piano evaluation is based primarily on VRChat OSC response logs, not direct evaluation of actual audio output
- External MIDI input is supported for verification purposes; actual audio is not natively output as MIDI by VRChat
- `extra_note_rate` may include duplicate reactions from VRChat OSC echo and is handled conservatively in correction and score updates
- Source code, character prompts, and personal memory data are not public
