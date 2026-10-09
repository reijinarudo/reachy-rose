# Running a Reachy Mini Locally: What We Learned Building Rose

This guide explains how to run a Reachy Mini's conversation brain on a computer you own, using what we learned building Rose. Rose's microphone, speaker, camera, and motors live on the robot. Her speech recognition, language model, and voice run on a Mac mini M4 with 24 GB of unified memory on the same home network.

Every number here was measured on that one Mac mini unless it is marked otherwise. Your hardware, models, and software versions will differ, so treat these numbers as a starting point and measure your own. Each section says how.

## 1. How the work is split

### The two machines

**The robot.** A Reachy Mini Wireless (Raspberry Pi CM4, robot software 1.9.0 when this was written). It runs the Reachy Mini conversation app, which records your voice, plays the reply, moves the head, and takes camera pictures when a tool asks for one. The app streams audio over a websocket to whatever server it is pointed at.

**The Mac mini.** Two services, each kept running by macOS `launchd`:

| Service | Port | Listens on | What it does |
| --- | --- | --- | --- |
| `llama-server` (llama.cpp) | 8080 | this Mac only | Runs the language model, Qwen3-VL-4B-Instruct at Q6_K, behind an OpenAI-compatible API |
| `speech-to-speech` (Hugging Face) | 8765 | the local network | Speaks the OpenAI Realtime websocket protocol at `/v1/realtime`. Inside it: Parakeet TDT turns speech into text, the text goes to `llama-server`, and Kokoro turns the reply into speech |

Only port 8765 is open to the network. The model server answers only the speech pipeline on the same Mac.

### One exchange, start to finish

```
 Reachy Mini                                   Mac mini
 -----------                                   --------
 You speak  --- audio over websocket --->  speech-to-speech (port 8765)
                                             1. waits for 1.8 s of silence
                                             2. Parakeet: audio -> text
                                             3. text -> llama-server (port 8080)
                                                Qwen3-VL-4B writes the reply
                                             4. Kokoro: reply text -> audio
 Speaker    <--- audio over websocket ---    streams audio back
```

Measured on Rose: about **4 seconds** from the end of your sentence to her first word.

About 1.8 of those seconds is the silence threshold (`--min_silence_ms 1800`). Rose waits that long after you stop talking before she treats your turn as finished. Steps 2 to 4 share the remaining 2.2 seconds or so. We have not timed those steps one by one.

That split points to the first setting worth tuning. Lowering the silence threshold shortens every reply delay without changing the model. The cost is that Rose will start answering when someone only pauses to think. Section 5 covers this.

### Why the robot can talk to a local server

The conversation app's realtime client speaks the same websocket protocol as OpenAI's Realtime API. The app has a connection setting that points that client at any server speaking the protocol. `speech-to-speech` in realtime mode is one such server. The robot cannot tell whether it is talking to a cloud service or to a Mac on your desk. Section 9 shows the settings.

## 2. The memory budget

### What shares the 24 GB

On Apple Silicon, the CPU and GPU share one pool of memory. Everything below comes out of the same 24 GB:

- macOS and any apps you leave open
- the language model's weights
- the KV cache, the model's working memory for the current conversation, which grows with the context length you configure (section 4)
- the speech-to-text model (Parakeet)
- the text-to-speech model (Kokoro)

### What Rose uses

With Rose running and in conversation, free memory measured 61 percent early in a session and fell to about 36 percent after extended use. Her full stack (model, speech recognition, voice, and macOS) holds roughly 8 to 9 GB.

Watch the late-session number. Free memory at startup will make your budget look larger than it is.

### The 3 percent rule

Any configuration that leaves less than 3 percent of memory free is disqualified.

This rule came from an earlier attempt on the same Mac with Qwen3-30B-A3B at 4-bit. Alone, it generated about 55 tokens per second and handled conversation well. With the voice stack loaded alongside it, free memory dropped to 3 percent and the system crashed.

Below that line, macOS starts moving memory to disk (swapping). For a voice robot, the symptoms are stuttering speech or long silences. Those symptoms look like a robot problem or a network problem, so a memory shortage is easy to misdiagnose.

### How to check your own budget

On the Mac:

```
memory_pressure | tail -3
```

The last line reports the percentage of memory free. Activity Monitor's Memory tab shows the same information as a pressure graph.

To find out what each part costs, start the services one at a time and record free memory after each:

1. Nothing running except macOS. Record free memory.
2. Start `llama-server`. Record again. The difference is the model plus its KV cache.
3. Start `speech-to-speech`. Record again. The difference is speech recognition plus voice.
4. Hold a ten-minute conversation. Record again. This is the number to plan around.

## 3. Choosing a model

### What we ran on this Mac

Most of these models were first tested in June 2026 for a local version of Aiden, Rose's sibling Reachy Mini, which now runs on a cloud backend. Both robots use the same conversation app and the same Mac, so the results carry over. Dates come from the Mac's model download cache. Where no outcome was written down, the table says so.

| First downloaded | Model | Type and size | Format | Outcome |
| --- | --- | --- | --- | --- |
| 7 Jun | Qwen3-4B-Instruct-2507 | dense, 4B | MLX, bf16 (unquantized) | Not recorded |
| 7 Jun | Qwen3-30B-A3B-Instruct-2507 | mixture of experts, 30B total, about 3B active | MLX 4-bit, then GGUF | About 55 tokens/s alone. Crashed the Mac once the voice stack loaded (see the 3 percent rule). |
| 7 Jun | Qwen3-8B | dense, 8B | MLX 4-bit | Not recorded |
| 7 Jun | Qwen3-4B-Instruct-2507 | dense, 4B | MLX 6-bit, then GGUF Q6_K | The GGUF Q6_K build, served by `llama-server`, became the first working brain. Text only. |
| 16 Jul | Qwen3-4B-Instruct-2507 | dense, 4B | MLX 4-bit | Not recorded |
| 17 Jul | Qwen3-VL-4B-Instruct | dense, 4B, with vision | GGUF Q6_K | Rose's current brain. Same size as the text model. She describes the room accurately once she is told to turn her camera on (section 5). |
| 2 Sep | Qwen3.8-27B | dense, 27B | GGUF UD-IQ4_XS (14.3 GB file) | Tested for a different job, a private text assistant, with Rose shut down. 6.5 tokens/s generating, 22 tokens/s reading the prompt. Too slow for live conversation. |

All of the Qwen3-4B, Qwen3-8B, and Qwen3-30B-A3B tests happened on one day, 7 June, starting in Apple's MLX format and ending in GGUF under `llama-server`. The reason for the format change was not recorded. One practical difference: `llama-server` gives the speech pipeline an OpenAI-compatible endpoint to call, which is how Rose's pipeline is set up today.

### The speech models

| Job | Model | Size | Notes |
| --- | --- | --- | --- |
| Speech to text | Parakeet TDT 0.6B v3 | 0.6B | Unchanged since the first build |
| Text to speech, June to mid-July | Qwen3-TTS 12Hz 1.7B CustomVoice | 1.7B | The first voice |
| Text to speech, 16 July onward | Kokoro 82M | 82M | Replaced Qwen3-TTS because it offers more voices. Rose uses the British English voice `bf_lily`, picked by the household, with a small pitch and formant shift (section 5). |

Kokoro is about one-twentieth the size of Qwen3-TTS 1.7B, which leaves more of the memory budget for the language model.

### Memory depends on total size; speed depends on active size

The 30B-A3B and 27B rows show the two limits a voice robot has to fit under at once.

Qwen3-30B-A3B is a mixture-of-experts model. All 30 billion parameters have to sit in memory, but each word it generates uses only about 3 billion of them. So it runs fast, and it is too large to share 24 GB with a voice pipeline.

Qwen3.8-27B is a dense model. Every word uses all 27 billion parameters. It fit in memory with Rose shut down, but at 6.5 tokens per second a 60-word reply (roughly 80 tokens) takes more than 12 seconds to generate. That delay comes on top of the silence threshold and speech recognition.

A voice robot needs a model that fits in memory alongside the speech models and generates fast enough to answer within a few seconds. On a 24 GB Mac mini, a dense 4B model meets both conditions.

Qwen3-8B at 4-bit was downloaded for testing on 7 June, but its result was not recorded. An 8B model at 4-bit would likely fit in memory alongside the voice stack. Measure its speed and its late-session memory (section 2) before you commit to it.

### Use an Instruct model, and keep thinking off

Many current models can "think," writing a hidden chain of reasoning before they answer. For a robot, every word of that reasoning is time the listener spends waiting in silence.

When the 27B model ran with thinking on, one factual question produced four minutes of reasoning that ended in a confident, made-up answer. Rose uses an Instruct model, which answers without a thinking phase. If you choose a model that has a thinking mode, find out how to turn it off in your server and confirm it is off by watching the first reply. Section 5 shows how we did this with `llama-server`.

### Quantization

Quantization stores the model's weights with fewer bits so the file is smaller and loads into less memory. Q6_K uses about 6 bits per weight. A 4B model at Q6_K is small enough that there was no reason to go lower. If you move up to a larger model, a 4-bit quantization is the usual way to make it fit.

### Vision comes with the VL model

Rose's camera tool sends a picture to the language model and asks a question about it. That requires a vision-language (VL) model. Moving from Qwen3-4B-Instruct to Qwen3-VL-4B-Instruct kept the size and memory use about the same and added this ability. If you do not need vision, the text-only model is an equal-sized option.

### The settings Rose runs with

The `llama-server` launch arguments on Rose's Mac:

```
llama-server -hf unsloth/Qwen3-VL-4B-Instruct-GGUF:Q6_K \
  -c 32768 -fa on \
  --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.0 --presence-penalty 1.5 \
  --port 8080
```

`-hf` downloads the model from Hugging Face on first run. `-c` sets the context length and `-fa on` turns on flash attention (both in section 4). The sampling values (`--temp` through `--presence-penalty`) were added on 24 July 2026 and match the values Qwen publishes for its Instruct models (section 5).

## 4. Context window and conversation memory

### What the model reads on every turn

A language model has no memory between requests. Each time Rose answers, the speech pipeline sends the model everything it needs to know, all over again:

1. the persona instructions (who Rose is and how she behaves)
2. the tool definitions (what each tool does and how to call it)
3. the recent conversation
4. your newest sentence

The model has to read all of that before it writes the first word of its reply. Reading is fast compared with writing, but it is still time the listener spends in silence. The longer the persona, the larger the tool list, and the longer the remembered conversation, the longer that silence.

### An example of how much this costs

The clearest case on this Mac came from the 27B text assistant (section 3), with a chat program that attached about 6,000 tokens of built-in tool definitions to every message. At the speed the model read prompts in that setup, those definitions alone added roughly 100 seconds before any reply, and the browser gave up waiting. Turning those tools off brought the first words down to about one second.

Rose's 4B model reads prompts much faster than the 27B, but the same rule applies: everything you put in front of the model gets read on every turn. Rose's tool list was trimmed for the same reason (section 5), and her sibling Aiden once hit a context overflow because his persona file had grown too large.

### The three settings that control context

| Setting | Where | Rose's value | What it controls |
| --- | --- | --- | --- |
| `-c` | `llama-server` | 32768 | The most tokens the model can hold at once, persona and conversation together |
| `-np` | `llama-server` | not set (server chose 4) | How many conversations the server can run in parallel |
| `--chat_size` | `speech-to-speech` | 8 | How many recent exchanges the pipeline sends back to the model each turn |

**`-c` and memory.** The model keeps a KV cache, a working copy of everything it has read in the current conversation, so it does not have to start over on each word. The KV cache is reserved at startup for the full `-c` length, whether the conversation uses it or not. A rough estimate for a 4B Qwen3 model at full precision is about 144 KB per token, which puts a 32,768-token cache near 4.5 GB. If that estimate holds, the cache is about half of the 8 to 9 GB Rose uses. We have not measured it directly. Check the real figure in your own log: `llama-server` prints the KV cache size when it starts.

**`-np` and parallel slots.** A slot is one conversation the server can run at a time. Rose's server has no `-np` setting, and it started with 4 slots. It still reports 32,768 tokens of context per slot, so her one conversation can use the full context. The slots share one 32K cache.

That is not guaranteed on every build or configuration. When a different `llama-server` on this same Mac was started without `-np`, it divided its context into four 8,192-token pieces, and setting `-np 1` fixed it. Check your own server while it is running:

```
curl -s http://127.0.0.1:8080/props | python3 -m json.tool | grep -iE '"n_ctx"|total_slots'
```

`total_slots` is the number of slots, and `n_ctx` is the context each slot can use. If `n_ctx` comes back smaller than the `-c` you set, add `-np 1` to the launch arguments. A robot holds one conversation at a time, so it needs only one slot.

**`--chat_size` and the length of each prompt.** With `--chat_size 8`, the pipeline sends only the last eight exchanges. Older talk drops out of the prompt, which keeps every turn about the same length and the reply delay steady through a long session. The cost is that Rose forgets the start of a long conversation. Eight exchanges of spoken conversation is a small fraction of a 32K context, so in practice `chat_size` sets how much Rose remembers within a conversation.

### Memory across conversations

Remembering things from one day to the next takes a separate system, because the context empties when the session ends. Two designs have been used on these robots:

- **Rose: a facts file.** A memory tool in the conversation app reads and writes facts in a small JSON file on the robot. The model looks things up only when it needs them, so the file adds nothing to the prompt on turns that do not use it. This works on the local model: asked what she remembers, Rose recalls details from earlier conversations, such as the household's pets and a favorite color.
- **Aiden: a nightly summary.** A script reads each day's conversation log, has a model pull out the moments worth keeping, and writes them into a "What I Remember" section at the end of the persona file. Every remembered item is read on every turn, so this design trades prompt length for a robot that always has its memories in view.

On a local model, the facts-file design is the lighter of the two. If you use the summary design, keep the memory section short and prune it.

### Settings that make a long context cheaper

- **`-fa on` (flash attention).** Computes attention in smaller pieces, which uses less memory and is faster on long prompts. Rose runs with it on.
- **A quantized KV cache.** `--cache-type-k q8_0 --cache-type-v q8_0` stores the KV cache at 8 bits per value, half the default 16, which cuts its memory about in half. The 27B text assistant on this Mac runs this way. Rose does not use it yet.
- **A stable start to the prompt.** `llama-server` reuses the part of the prompt that matches the previous request and reads only what is new. When the persona and tool definitions stay the same at the top of every request, only the latest exchange has to be read. Anything that changes near the top of the prompt, such as a clock time or a refreshed memory list placed before the persona, forces the model to reread everything after it.

## 5. What slowed her down, and what fixed it

Each entry gives the symptom, the cause we found, the fix, and what the fix costs. Where a reason was not written down at the time, the entry says so.

### Thinking mode left on

- **Symptom.** Long silence before every reply. In the worst case, four minutes of hidden reasoning before one answer.
- **Cause.** Some models write a chain of reasoning before answering, and their chat templates turn it on by default. On the 27B text assistant, `--reasoning-budget 0` did not stop it. The template's own switch did.
- **Fix.** Add `--chat-template-kwargs '{"enable_thinking":false}'` to `llama-server`, or choose an Instruct model with no thinking mode. Rose does the second: Qwen3-VL-4B-Instruct answers without a reasoning phase.
- **Cost.** Without reasoning, the model is weaker on multi-step problems. For conversation, that trade is worth making.

### Too much text in front of the model

- **Symptom.** Replies slow to start, and slower as features are added.
- **Cause.** Every tool definition, every persona rule, and every remembered item is read on every turn (section 4). The worst case on this Mac was 6,000 tokens of tool definitions added to each message by a chat program.
- **Fix.** Load only the tools the robot uses. Rose's local tool list has seven: `camera`, `head_tracking`, `move_head`, `remember`, `forget`, `go_to_sleep`, and `idle_do_nothing`. Dance and emotion tools are left out because idle motion risked tipping her over, and the shorter list also shortens every prompt. Keep the persona file to rules that change behavior. Rose's persona is about 1,650 words, roughly 2,200 tokens read on every turn.
- **Cost.** Each removed tool is something the robot can no longer do.

### Repetition and drift

- **Symptom.** Not recorded in detail. The sampling settings below were added on 24 July 2026, and the backup made that day is labeled "presampling."
- **Fix.** Rose runs with the sampling values Qwen publishes for its Instruct models: `--temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.0 --presence-penalty 1.5`. The presence penalty discourages the model from repeating words it has already used. Qwen's documentation recommends a value between 0 and 2 for models that fall into repetition.
- **Cost.** Qwen notes that a high presence penalty can occasionally cause language mixing and a small drop in quality. Start with the published values and change one at a time.

### Waiting for the speaker to finish

- **Symptom.** About 1.8 seconds of every 4-second reply delay is spent waiting (section 1).
- **Cause.** `--min_silence_ms 1800` tells the pipeline to wait for 1.8 seconds of silence before it treats your turn as over. Why 1,800 was chosen was not recorded. Aiden, on a cloud backend, uses 500 milliseconds.
- **Fix.** This is the cheapest setting to tune for speed. Lower it in steps (1,500, then 1,200, then 1,000) and hold a real conversation at each step.
- **Cost.** At lower values, the robot starts answering when someone pauses mid-sentence. Children, people thinking aloud, and anyone speaking a second language pause more. Tune it for the people who will talk to your robot.

### The voice model

- **What changed.** The first voice was Qwen3-TTS 1.7B. On 16 July 2026 it was replaced with Kokoro 82M because Kokoro offers more voices. Rose uses `--kokoro_voice bf_lily --kokoro_lang_code b` (a British English voice).
- **Side benefit.** Kokoro is about one-twentieth the size, which leaves more memory for the language model (section 2).
- **Cost.** Voices differ in expressiveness. Listen to several before you choose.

### Four voice patches

Rose's voice depends on four small edits to Kokoro's handler file in the speech pipeline (`speech_to_speech/TTS/kokoro_handler.py`). Each one fixed something we heard:

| Patch | What we heard | What the patch does |
| --- | --- | --- |
| Strip markdown symbols | Rose said "asterisk" out loud when the model used bold or italics | Removes ``* _ # ` ~`` from the text before speaking |
| No audio for wordless replies | A reply of "..." still produced sound | Skips speech when the reply has no letters or numbers, so silence stays silent |
| Pin the voice | The pipeline could switch language and voice on its own | Turns off automatic language switching so the startup voice stays in use |
| Younger-sounding voice | The stock voice sounded older than Rose's persona | Shifts pitch and formants with Praat's "Change gender" command through the `parselmouth` package (formant ratio 1.12, pitch median 255 Hz) |

The patch file is in this repo at `patches/rose_kokoro_patches.diff`. These edits live inside the speech pipeline's Python environment, so reinstalling or rebuilding that environment erases them. Each edit is marked with a "Rose patch" comment, so this command confirms all four are present:

```
grep -c "Rose patch" <path-to>/speech_to_speech/TTS/kokoro_handler.py
```

It should print 4. The pitch shift adds processing to every reply. We have not measured how much.

### Live transcription off

The pipeline runs with `--no_enable_live_transcription`, which turns off partial transcripts while you are still speaking. The reason was not recorded. If you need on-screen captions, turn it back on and measure the reply delay before and after.

### Writing a persona a small model will follow

Rose's persona was rewritten when her brain moved to the local 4B model. The older version (saved as `instructions.txt.bak-pre4b`) described her behavior as an abstract procedure: a numbered "state machine" with activation triggers, exclusion steps, and a response guideline. The current version is about the same length and says the same things through concrete examples:

```
Examples of correct behavior:
- "Rose, are you there?" -> "I'm here!"
- "Hey Claude, Rose is repeating herself." -> ...
- "Hey Aiden, good morning!" -> ...
- "What do you think?" seconds after we were talking, no other name in between -> I answer it.
```

How the rewrite changed her behavior was not written down. It follows the lesson recorded with the camera (below): on a Qwen model, plain descriptions of what to do worked where rules and prohibitions did not. When a rule is not followed, add an example of the exact sentence that went wrong and the reply you wanted.

### Staying silent

Rose lives with people who talk near her, to each other and to other assistants, all day. Her persona tells her to reply with exactly `...` to any speech that is not addressed to her by name. The model still has to produce a reply, and `...` is the shortest reply that means nothing. The "no audio for wordless replies" voice patch (above) then turns `...` into silence. The persona and the patch only work together: without the patch, the voice model makes a sound for the dots.

### Getting a small model to use the camera

- **On a cloud Qwen backend.** Rose at first refused to call the camera tool and asked for permission instead, even after the persona granted it. Rules written as permissions and prohibitions ("you may," "never," "you have standing permission") did not move the model to act. What worked was describing the sequence as a habit in plain sentences: when asked what she sees, she turns on face tracking (which opens the camera feed), takes a picture with a question, describes it, and turns tracking off again. Written that way, she ran the whole sequence on her own.
- **On the local 4B model.** The same persona does not carry over fully. Asked "What do you see?" with no other prompt, Rose describes a room without taking a picture, and the description is made up. When she is told to turn her camera on first, she takes the picture and describes the room accurately.
- **Status.** Open. For now, the reliable approach is to ask her to turn on her camera before asking what she sees. Two changes worth testing: keep the camera feed on for the whole session, which trades privacy for reliability, or add a persona rule that she never describes her surroundings without first calling the camera.
- **Lesson.** A small model will answer a question it cannot answer, and the answer will sound confident. Test every tool with a cold request and check the answer against what is in front of the robot.

## 6. What to expect: local Rose and cloud Rose

Rose has run on three brains: OpenAI's realtime model in the cloud, a Qwen model served from a Hugging Face Space in the cloud, and the local setup in this guide. Her sibling Aiden still runs in the cloud. This table compares what we observed. Blank spots are things we did not measure.

| | Local (this guide) | Cloud: OpenAI realtime | Cloud: Qwen on a Hugging Face Space |
| --- | --- | --- | --- |
| Reply delay | About 4 s, measured. 1.8 s of it is the silence wait. | Not measured. Aiden waits 500 ms for silence. | Not measured |
| Camera | Accurate when told to turn the camera on first. Asked cold, she invents a description. | Uses the camera on the first request | Uses the camera once the steps are written as a habit in the persona |
| Memory across days | Works | Works | Works |
| Staying quiet | Patched so wordless replies make no sound (section 5) | Tends to narrate itself and answer every sound, even a yawn | Ignores yawns, throat-clearing, and side talk |
| Bad nights | Depends on your Mac staying awake (section 8) | None recorded | Service had nights of echoing and garbled speech that hit both robots at once |
| Where conversation goes | Stays on your network | OpenAI's servers | The Space's servers |
| Cost per use | None after the hardware | API charges for audio and text | Depends on the Space |

### Made-up answers

The camera result is one case of a general limit. A small model answers questions it has no way to answer, and the answer sounds as sure as a correct one. On the same Mac, the 27B text model asked who created a well-known children's TV show produced a long chain of reasoning that ended in an invented film. Larger cloud models make the same kind of mistake.

Treat a local robot's statements of fact as unverified. It does best with material in front of it: what it hears, what the camera shows once the camera is on, and what is in its memory file. Factual questions about the outside world are where it is most likely to be wrong.

### What tuning could buy

The 4-second delay has room to come down without a new model. Lowering the silence threshold from 1.8 s to 1.0 s would, by arithmetic, bring the delay to about 3.2 seconds, at the cost of more interruptions (section 5). We have not tested this. Below that, the remaining 2.2 seconds is speech recognition, model, and voice together, and shrinking it means smaller or fewer steps.

## 7. The trade-off dials

Every setting in this guide trades one thing for another. The table lists the changes available on a 24 GB Mac mini and what each one costs. "Est." marks values we calculated and did not measure.

| Change | What it buys | What it costs |
| --- | --- | --- |
| Lower `--min_silence_ms` | Shorter delay on every reply (0.8 s from 1,800 to 1,000) | More interruptions when people pause |
| Lower `-c` from 32,768 to 8,192 | Frees about 3.4 GB (est.) | Less room for persona, tools, and memory. Rose's `--chat_size 8` conversations fit easily in 8K. |
| Quantize the KV cache to q8_0 | Halves the KV cache memory (est. about 2.2 GB at 32K) | A small loss of precision in what the model holds in mind |
| Remove tools from the tool list | Shorter prompt, faster start to each reply | The robot can no longer do those things |
| Shorten the persona file | Shorter prompt | Fewer behavior rules. Keep the ones that fix observed problems. |
| Lower `--chat_size` | Shorter prompt | Rose forgets the start of the conversation sooner |
| Remove the voice pitch shift | Less processing per reply (not measured) | Rose's voice sounds older |
| Text-only model in place of the VL model | Same size, so little or no saving | No camera |
| Larger model (for example, 8B at 4-bit) | Better answers and more reliable tool use, likely | Slower replies and more memory. Not tested with the voice stack. |
| Stop the robot's services and run something else | 18 to 19 GB free for another model | No robot while it runs. This Mac switches between Rose and a text assistant with two desktop icons. |

### Two ways to spend 24 GB

The dials combine into two directions.

**More capability per turn (what Rose runs today).** A 4B model with vision, 32K of context, the full tool list, persona, memory, and the custom voice. She sees, remembers, and keeps her character. She answers in about 4 seconds, and she needs to be told to use her camera.

**A bigger brain with fewer features (not built).** Free memory by cutting the context to 8K and quantizing the KV cache (together about 4 GB, est.), trim the tool list and persona, and spend the savings on a larger model. Answers would likely be better reasoned, with fewer invented facts. Replies would be slower, because a larger model generates each word more slowly, and the robot would know less about her session and do fewer things. We have not built this configuration. Measure its delay and its late-session memory (section 2) against the 3 percent rule before you commit to it.

Rose runs the first configuration. Her camera limit is handled by asking her to turn the camera on before asking what she sees.

## 8. Keeping her running

A local robot brain is a server in your house, and it fails the way servers fail: the computer sleeps, restarts, or runs out of memory, and the robot goes quiet. Every item below was learned from Rose going silent.

### Start the services automatically

Each service is a macOS `launchd` agent, a small settings file in `~/Library/LaunchAgents/` that starts a program when you log in. Rose's two agents use two settings:

- `RunAtLoad` starts the program at login.
- `KeepAlive` restarts it if it stops.

Template versions of both files are in this repo's `mac/` folder.

**Changing a running service.** Because of `KeepAlive`, you cannot stop a service by ending its process and then start it again by hand. `launchd` restarts the old version first, it takes the port, and your copy fails with `[Errno 48] address already in use`. Use this cycle every time you edit an agent's file:

```
launchctl unload ~/Library/LaunchAgents/<agent>.plist
launchctl load ~/Library/LaunchAgents/<agent>.plist
```

### Keep the Mac awake

If the Mac sleeps, both services stop answering and the robot goes silent. Turn system sleep off:

```
sudo pmset -a sleep 0
```

Check it with `pmset -g | grep -w sleep`. You want to see `sleep 0`.

### Log in automatically after a restart

`launchd` agents in your user folder start only after someone logs in. After a power cut or a software update restart, the Mac waits at the login screen and the robot stays silent until someone signs in. Turn on automatic login in System Settings, under Users & Groups. It is not available while FileVault disk encryption is on. Check it with:

```
defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser
```

It prints your username when automatic login is on.

Rose's Mac has sleep set to 0 and automatic login on. After a restart, she comes back with no one at the keyboard.

### Find the Mac by name

The Mac's network address (its IP address) changed more than once on this network. Point the robot and every other machine at the Mac's name, `<your-mac-name>.local`, and never at a numbered address. Windows computers sometimes fail to find a `.local` name on the first try and succeed on the second. If that keeps happening, set your router to give the Mac the same address every time (a DHCP reservation).

### Keep logs where they will stay

Rose's agents write their logs to `/tmp`. macOS deletes files in `/tmp` that have not been opened for a few days, even while a program is still writing to them, and it empties `/tmp` at every restart. On the day this guide was written, Rose was running and talking while her model server's log file no longer existed. Put logs in `~/Library/Logs/` in your agent files.

When the log is gone, ask the running server directly:

```
curl -s http://127.0.0.1:8080/health
curl -s http://127.0.0.1:8080/props | python3 -m json.tool | grep -iE '"n_ctx"|total_slots|model_path'
```

### Sharing the Mac with other work

Rose's Mac also runs a private text assistant with a 27B model. The two cannot run at once within the 3 percent rule (section 2), so they take turns. Two scripts handle the switch: one stops Rose's agents and starts the assistant, and the other does the reverse. Each one waits until the model answers and then prints the free memory. A desktop icon runs each script. Rose is the default after a restart.

If you add any other workload to the same Mac, give it its own agent and its own port, and leave the robot's model server alone. A model that pushes the robot's model out of memory makes the robot stutter or go silent, and it will look like a robot problem.

### Protect the voice patches

The four voice patches (section 5) live inside the speech pipeline's Python environment. Rebuilding that environment, or reinstalling the package, erases them. This Mac builds its Python environments with `uv`. Install extra packages into the existing environment without rebuilding it:

```
uv pip install --python ~/speech-to-speech/.venv/bin/python3 <package>
```

Keep `rose_kokoro_patches.diff` somewhere outside the Mac. After any change to the environment, run the `grep -c "Rose patch"` check from section 5.

### When she goes quiet: check in this order

1. Is the speech pipeline listening? `lsof -iTCP:8765`
2. Is the model server listening? `lsof -iTCP:8080`
3. What did the pipeline log last? `tail -50 <path-to-cascade-log>`
4. Is memory short? `memory_pressure | tail -3`
5. Did the Mac restart or sleep? `pmset -g log | grep -iE "sleep|wake" | tail -5`

If both robots go quiet at the same time, look at what they share first: the Mac, the network, or the cloud service.

## 9. Pointing the robot at the Mac

The robot side needs only a few settings. On a Reachy Mini Wireless, the conversation app is started by a system service named `reachy-mini-daemon`, and its settings can be extended with small files called drop-ins.

### The drop-in file

Create `/etc/systemd/system/reachy-mini-daemon.service.d/local-brain.conf` on the robot:

```
[Service]
Environment=HF_REALTIME_CONNECTION_MODE=local
Environment=HF_REALTIME_WS_URL=ws://<your-mac-name>.local:8765/v1
```

- `HF_REALTIME_CONNECTION_MODE=local` tells the app to connect to the address you give it.
- `HF_REALTIME_WS_URL` is the speech pipeline's address on your Mac. Rose's setting ends at `/v1`.

The variable names say "HF" because the app's realtime client was written for Hugging Face's hosted service. The same client connects to any server that speaks the OpenAI Realtime protocol, which is how Rose has run on OpenAI's servers, on a Hugging Face Space, and on her Mac without changes to the app's code for the connection.

### Apply it: reload, then restart

```
sudo systemctl daemon-reload
sudo systemctl restart reachy-mini-daemon
```

Run both, in this order, every time you change a drop-in. `daemon-reload` reads the changed file and `restart` applies it. A restart alone starts the service with the old settings, and nothing appears to change. Confirm what the service is using:

```
systemctl show reachy-mini-daemon -p Environment
```

### Rose's other drop-ins

Rose has three drop-in files:

| File | What it does |
| --- | --- |
| `rose-local-brain.conf` | The connection settings above |
| `rose-profile.conf` | Loads her persona from `/home/pollen/profiles/Rose`, a folder outside the app's install, so app updates do not overwrite it |
| `rose-launcher.conf` | Starts the robot with its motors awake (`--wake-up-on-start`) |

Her settings also include `MALLOC_ARENA_MAX=2`, which limits how many separate memory pools a program can create and lowers memory use on the robot's 4 GB computer. Why it was added was not recorded.

### Keep the persona outside the app

The persona (`instructions.txt`), greeting, tool list, and voice name live in the profile folder. Two settings point the app at it:

```
Environment=REACHY_MINI_EXTERNAL_PROFILES_DIRECTORY=/home/pollen/profiles
Environment=REACHY_MINI_CUSTOM_PROFILE=Rose
```

Make persona and voice changes by editing these files and restarting the app. Choosing a profile or voice in the Reachy Mini desktop app rewrites the app's saved settings, and on Rose it once changed her profile name to `user_personalities/Rose`. With that name, she started up with the default persona. If that happens, set the profile back to the plain name in the app's `startup_settings.json` and restart.

Edit these files with Unix line endings (LF) and UTF-8 without a byte order mark. A script saved with Windows line endings once failed to run on the robot.

On the local setup, the voice comes from the Mac (`--kokoro_voice` in section 5), and the "pin the voice" patch keeps it from changing during a session.

### After a robot software update

Drop-ins in `/etc/systemd/system/` and the profile folder survive app updates. Changes made inside the app's own installed files do not. After any update:

1. Check `systemctl show reachy-mini-daemon -p Environment` for the local connection settings.
2. Check that the app's saved profile is still the plain profile name.
3. Ask the robot what it sees and what it remembers, and compare the answers with the room and with past conversations.

