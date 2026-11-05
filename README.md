# Logic Pro MCP

An MCP server that lets Claude write and play music in Logic Pro.

**Stage 1 (this version): MIDI.** Claude can play notes, chord progressions, and drum patterns live into Logic, or export them as `.mid` files.
Coming next: transport and track setup (Stage 2), mixer control (Stage 3).

## How it works

```
Claude Desktop ──stdio──▶ logic-pro-mcp (Python) ──IAC Driver──▶ Logic Pro
                                   └──────▶ ~/Music/Logic MCP Exports/*.mid
```

- Logic has no public API, so the server talks to it the way a MIDI keyboard would: through macOS's built-in virtual MIDI cable, the **IAC Driver**.
- Claude Desktop launches the server as a subprocess and speaks JSON-RPC over stdin/stdout. That's why the code never `print()`s to stdout; logs go to stderr.

## Tools

| Tool | What it does |
|---|---|
| `play_notes` | Any note sequence: melodies, basslines, custom voicings |
| `chord_progression` | Chord symbols (`Am7`, `Fmaj7`, `G/B`) as sustained chords, beat stabs, or offbeat house stabs |
| `drum_pattern` | Step-sequencer strings like `x...x...x...x...` per drum, with swing |
| `list_midi_ports` | Shows MIDI outputs and the export folder (for troubleshooting) |
| `panic` | All notes off |

Every music tool takes `destination`: `"play"` (live into Logic), `"file"` (save a `.mid`), or `"both"`.

Note names follow Logic's convention: **C3 = middle C** (MIDI 60).

## Setup (macOS)

### 1. Turn on the IAC Driver
1. Open **Audio MIDI Setup** (Spotlight it).
2. **Window → Show MIDI Studio**.
3. Double-click **IAC Driver** and check **Device is online**. Click Apply.

### 2. Install the server
```bash
git clone https://github.com/aribashaikhhh/logic-pro-mcp.git
cd logic-pro-mcp
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
python -m pytest -q tests        # optional: should say 8 passed
which logic-pro-mcp              # copy this path for the next step
```

### 3. Register it with Claude Desktop
Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (create it if missing):

```json
{
  "mcpServers": {
    "logic-pro": {
      "command": "/ABSOLUTE/PATH/TO/logic-pro-mcp/.venv/bin/logic-pro-mcp"
    }
  }
}
```

Use the absolute path from `which`: Claude Desktop doesn't load your shell's PATH. Fully quit and reopen Claude Desktop afterwards.

Optional environment variables (add an `"env": {...}` block):
- `LOGIC_MCP_EXPORT_DIR`: where `.mid` files go (default `~/Music/Logic MCP Exports`)
- `LOGIC_MCP_PORT`: MIDI port name to match (default `IAC`)

### 4. Set up Logic
- Create a **Software Instrument** track and select it. Logic sends incoming MIDI from any input to the selected track.
- For drums, load **Drum Machine Designer** (Electronic Drum Kit) or **Drum Kit Designer**.
- To capture what Claude plays: arm the track, press **R** to record, then ask Claude to play. Set Logic's tempo to match the `bpm` you ask for.
- Exported `.mid` files: drag them from Finder into the tracks area.

## Try it

> "Play a four-on-the-floor house beat at 124 BPM with offbeat open hats and a clap on 2 and 4."

> "Export an Am7 – Fmaj7 – C – G progression as offbeat house stabs, 4 repeats, to a file called verse-chords."

> "Write a syncopated bassline in A minor that follows those chords and save it as bass."

## Security notes

- The server only does two things: send MIDI to one local port, and write `.mid` files to one folder.
- Filenames are sanitized and the final path is checked to stay inside the export folder, so a request like `../../.zshrc` can't write elsewhere (see `tests/test_theory.py`).
- Playback is capped at 180 seconds per call, and an all-notes-off is always sent afterwards so nothing gets stuck.
- Stage 1 needs **no** Accessibility permissions. Later stages will, and they'll be scoped and documented when added.

## Troubleshooting

| Problem | Fix |
|---|---|
| "No MIDI output matching 'IAC'" | IAC Driver isn't online (step 1) |
| "MIDI backend missing" | `pip install python-rtmidi` inside the venv |
| Tools don't appear in Claude | Check the absolute path, then fully quit/reopen Claude Desktop. Logs: `~/Library/Logs/Claude/mcp-server-logic-pro.log` |
| Notes play but no sound | Select a Software Instrument track in Logic |
| Notes are stuck | Ask Claude to run `panic` |

## Roadmap

- [x] **Stage 1:** MIDI: notes, chord progressions, drum patterns, `.mid` export
- [ ] **Stage 2:** Transport (play, stop, record, tempo) and track setup
- [ ] **Stage 3:** Mixer control (volume, pan, mute, solo) via Mackie Control / OSC

## License

PolyForm Noncommercial 1.0.0. Free for personal and noncommercial use.
For commercial use, contact me. See [LICENSE](LICENSE).
