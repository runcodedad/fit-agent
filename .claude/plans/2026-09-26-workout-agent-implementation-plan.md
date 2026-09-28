# Garmin Workout Agent — Implementation Plan

> Written 2026-09-26, when this project was still at `/home/nick/Code/runcodedad/garmin` (since renamed to `fit-agent`). Produced by a planning subagent after two research passes: one inventorying the (then-empty) project directory, and one doing deep source-reading of the Garmin FIT C# SDK checked out at `/home/nick/Code/fit-csharp-sdk`.

Research complete. Both the Garmin FIT SDK internals and the Anthropic C# SDK surface are now verified directly from source (not just from the summarized ground truth), which surfaced one important correction to the original brief (detailed in section 4). Here is the full implementation plan.

## 0. Key correction to the brief (found by reading source, not just the Cookbook comments)

The prior research said "HR custom values offset by +100, power by +1000, speed scaled by 1000" as if these were uniform SDK-level transforms. Reading `/home/nick/Code/fit-csharp-sdk/Dynastream/Fit/Profile.cs` (`CreateWorkoutStepMesg()`, lines ~4600-4630) and `FieldBase.cs` (`SetValue`/`GetValue`) shows two genuinely different mechanisms:

- **Speed is a true SDK scale.** The `CustomTargetSpeedLow`/`High` subfield is defined with `scale=1000, offset=0, units="m/s"`. The typed setter `SetCustomTargetSpeedLow(float metersPerSecond)` automatically multiplies by 1000 before writing the raw `uint32`. **Do not manually multiply by 1000 — call the typed setter with plain m/s and let the SDK do it.**
- **HR and Power are not SDK scale/offset at all — they're a wire-format convention the SDK does not enforce.** The `CustomTargetHeartRateLow`/`High` subfield is defined with `scale=1, offset=0`. `SetCustomTargetHeartRateLow(uint)` writes the raw value unchanged. The "+100" is a *value-range convention* interpreted by the watch itself: raw value `0–100` means "% of max HR", raw value `>=100` means "bpm, where actual bpm = raw − 100". The application (our mapping layer) is responsible for adding 100 when the target is an absolute bpm value, not the SDK. Same story for power: raw `<1000` = %FTP, raw `>=1000` = watts+1000, and the app must add 1000 itself. Duration `SetDurationTime(float seconds)` (scale 1000 → ms) and `SetDurationDistance(float meters)` (scale 100 → cm) are true SDK scales like speed, confirmed by the same Profile.cs table and matching the Cookbook's raw distance values (e.g. 80000 cm = 800 m).

This matters directly for v1 since pace/HR targets are explicitly in scope — the mapping layer must encode this HR convention explicitly rather than trusting a typed setter to "just work," and this needs an explicit unit test.

Also confirmed: `Decode` + `FitListener` (`decoder.MesgEvent += fitListener.OnMesg; decoder.Read(stream); var msgs = fitListener.FitMessages;`, exposing `.WorkoutMesgs`, `.WorkoutStepMesgs`, `.FileIdMesgs`) is the pattern for the round-trip verification test, per `/home/nick/Code/fit-csharp-sdk/Examples/Decode/DecodeDemo.cs`.

## 1. Project structure

```
/home/nick/Code/runcodedad/fit-agent/
  Fit.Agent.sln
  README.md                                 (usage + manual watch-transfer instructions)
  .gitignore                                (bin/, obj/, *.fit test outputs, .env)
  src/
    Fit.Agent.Core/
      Fit.Agent.Core.csproj                  (net10.0; PackageReference Garmin.FIT.Sdk 21.217.0, Anthropic *)
      Model/
        WorkoutPlan.cs
        WorkoutStep.cs
        DurationSpec.cs
        TargetSpec.cs
        RepeatBlock.cs
        StepKind.cs                          (enum: Warmup, Interval, Rest, Cooldown, RepeatStart/RepeatEnd marker)
      Interpretation/
        IWorkoutInterpreter.cs
        ClaudeWorkoutInterpreter.cs
        WorkoutSchema.cs                     (the JSON schema sent as OutputConfig.Format)
        WorkoutInterpreterOptions.cs         (model id, api key resolution, etc.)
      Mapping/
        IFitWorkoutMapper.cs
        FitWorkoutMapper.cs                  (WorkoutPlan -> WorkoutMesg + List<WorkoutStepMesg>)
        FitTargetEncoder.cs                  (isolates the HR/power +100/+1000 convention + speed passthrough)
      Encoding/
        IFitWorkoutFileWriter.cs
        FitWorkoutFileWriter.cs              (FileId -> Workout -> Steps -> Close pipeline)
      WorkoutAgent.cs                        (facade: NL string -> .fit bytes/file, composes the above)
    Fit.Agent.Cli/
      Fit.Agent.Cli.csproj                   (net10.0, OutputType Exe, ProjectReference to Core)
      Program.cs
  tests/
    Fit.Agent.Core.Tests/
      Fit.Agent.Core.Tests.csproj            (net10.0, xUnit, PackageReference Garmin.FIT.Sdk for decode-based assertions)
      Mapping/
        FitWorkoutMapperTests.cs             (fixed WorkoutPlan fixtures -> assert exact WorkoutStepMesg field values)
        FitTargetEncoderTests.cs             (unit tests for the HR/power convention specifically)
      Encoding/
        FitWorkoutFileWriterRoundTripTests.cs (encode -> Decode -> assert structural correctness)
        SampleWorkouts.cs                     (shared fixed WorkoutPlan builders: "800s repeats", "simple tempo run")
```

Rationale for `src/`+`tests/` split and `Core`/`Cli` naming: keeps the class library dependency-free of console concerns (per decision #2 — no `Console.ReadLine`/`Console.WriteLine` inside `Core`; all I/O happens in `Cli`), and leaves an obvious slot for a future `Fit.Agent.Api` (ASP.NET Core minimal API) sibling under `src/` that references `Core` the same way `Cli` does — nothing in `Core`'s public surface should assume a synchronous console (use `async Task<...>` throughout so a future controller can `await` the same facade).

`Fit.Agent.Core.csproj` key elements:
```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="Garmin.FIT.Sdk" Version="21.217.0" />
  <PackageReference Include="Anthropic" Version="*" />
</ItemGroup>
```
(Pin `Garmin.FIT.Sdk` to `21.217.0` rather than `*` — the reference source checked out locally is exactly that profile version, and pinning avoids silent behavior drift in the scale/offset tables just analyzed. Anthropic SDK can float since only stable, documented surface is used. `TargetFramework=net10.0` implies `LangVersion=14` by default — no explicit `<LangVersion>` element needed; the local machine has the .NET 10 SDK (`10.0.100`) installed alongside 8.0.404/9.0.101, so no SDK install step is required before scaffolding.)

Confirmed from the local checkout (`/home/nick/Code/fit-csharp-sdk/FitSDK.csproj`): the SDK multi-targets `netcoreapp2.0;net46;netstandard2.0`, no `net10.0`-specific target. This is not a blocker — `netstandard2.0` is fully consumable from a `net10.0` project — but it does mean the SDK's own binary was built/tested against much older runtimes, so keep the round-trip `Decode` integration test (§7.3) as the real gate on cross-runtime correctness rather than assuming NuGet restore succeeding is sufficient proof.

## 2. Domain model (plain C#, FIT-agnostic)

```csharp
public sealed class WorkoutPlan
{
    public string Name { get; init; }
    public string? Description { get; init; }
    public List<WorkoutItem> Items { get; init; } = new(); // flat ordered list; repeats are a bracketing item
}

public enum StepKind { Warmup, Work, Rest, Cooldown }

public sealed class WorkoutStep
{
    public StepKind Kind { get; init; }
    public string? Name { get; init; }
    public DurationSpec Duration { get; init; }
    public TargetSpec? Target { get; init; }
}

public sealed class RepeatBlock
{
    public int Repetitions { get; init; }
    public List<WorkoutStep> Steps { get; init; } = new(); // typically [Work, Rest]
}

// WorkoutItem is either a single WorkoutStep or a RepeatBlock — model as
// a discriminated union via an abstract base + two sealed subtypes rather
// than nullable dual fields, so the mapper can pattern-match exhaustively.
public abstract class WorkoutItem { }
public sealed class SingleStepItem : WorkoutItem { public required WorkoutStep Step { get; init; } }
public sealed class RepeatItem : WorkoutItem { public required RepeatBlock Repeat { get; init; } }

public abstract class DurationSpec { }
public sealed class TimeDuration : DurationSpec { public required double Seconds { get; init; } }
public sealed class DistanceDuration : DurationSpec { public required double Meters { get; init; } }
public sealed class OpenDuration : DurationSpec { } // "lap button" / until manually advanced

public abstract class TargetSpec { }
public sealed class PaceRangeTarget : TargetSpec
{
    public required double FastSpeedMetersPerSecond { get; init; }  // faster end of range
    public required double SlowSpeedMetersPerSecond { get; init; } // slower end of range
}
public sealed class HeartRateZoneTarget : TargetSpec { public required int Zone { get; init; } } // 1-5
public sealed class HeartRateRangeTarget : TargetSpec
{
    public required int LowBpm { get; init; }
    public required int HighBpm { get; init; }
}
```

This is exactly the JSON shape Claude is asked to return (see §3). Keeping `Fast/SlowSpeedMetersPerSecond` rather than "pace" strings avoids parsing min/km or min/mile in the mapper — push that conversion into the interpreter's schema/prompt instructions (ask Claude to always return m/s) since Claude is already doing the pace-lookup/estimation work (e.g., converting "5k pace" into a speed) and .NET should not have to guess a runner's 5k time.

## 3. Claude integration (`IWorkoutInterpreter`)

```csharp
public interface IWorkoutInterpreter
{
    Task<WorkoutPlan> InterpretAsync(string naturalLanguageDescription, CancellationToken ct = default);
}
```

`ClaudeWorkoutInterpreter` implements this using **structured outputs** (`OutputConfig.Format = new JsonOutputFormat { Schema = ... }`), not tool-use/forced-tool-choice — per the C# skill reference this is the recommended mechanism for "get JSON back reliably," and it sidesteps the fact that forced `tool_choice` is being deprecated/rejected on newer model families. One call, no agent loop needed since this is pure extraction (single LLM call tier, not an agent).

Schema (`WorkoutSchema.cs`) mirrors the domain model 1:1 — steps as a flat array where a repeat is `{"type":"repeat","repetitions":N,"steps":[...]}` and a plain step is `{"type":"step","kind":"warmup|work|rest|cooldown","duration":{...},"target":{...}}`. Sketch:

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "items": {
      "type": "array",
      "items": {
        "oneOf": [
          { "$ref": "#/$defs/step" },
          { "$ref": "#/$defs/repeat" }
        ]
      }
    }
  },
  "required": ["name", "items"],
  "$defs": {
    "duration": {
      "oneOf": [
        { "properties": { "type": {"const":"time"}, "seconds": {"type":"number"} }, "required":["type","seconds"] },
        { "properties": { "type": {"const":"distance"}, "meters": {"type":"number"} }, "required":["type","meters"] },
        { "properties": { "type": {"const":"open"} }, "required":["type"] }
      ]
    },
    "target": {
      "oneOf": [
        { "properties": { "type": {"const":"pace"}, "fastSpeedMps": {"type":"number"}, "slowSpeedMps": {"type":"number"} } },
        { "properties": { "type": {"const":"hrZone"}, "zone": {"type":"integer","minimum":1,"maximum":5} } },
        { "properties": { "type": {"const":"hrRange"}, "lowBpm": {"type":"integer"}, "highBpm": {"type":"integer"} } },
        { "properties": { "type": {"const":"none"} } }
      ]
    },
    "step": { "properties": { "type": {"const":"step"}, "kind": {"enum":["warmup","work","rest","cooldown"]}, "name":{"type":"string"}, "duration": {"$ref":"#/$defs/duration"}, "target": {"$ref":"#/$defs/target"} }, "required":["type","kind","duration"] },
    "repeat": { "properties": { "type": {"const":"repeat"}, "repetitions": {"type":"integer","minimum":1}, "steps": {"type":"array","items":{"$ref":"#/$defs/step"}} }, "required":["type","repetitions","steps"] }
  }
}
```

(Nested `$ref`/`oneOf` support depends on what Claude's structured-output validator accepts — if `oneOf`/`$ref` prove unsupported in practice, fall back to a flatter schema with a `type` discriminator field and all possible properties present-but-optional, and validate/reject at the C# deserialization layer instead. Flag this as a build-time risk to de-risk early with a throwaway console spike before wiring the full mapper.)

System prompt instructions should explicitly state: convert pace descriptions ("5k pace", "7:30/mile") to `fastSpeedMps`/`slowSpeedMps` using reasonable equivalent-effort estimation, always emit warmup/cooldown as `kind` even if the user says "10 minute warmup and cooldown" (split into two separate step objects), and represent "6x800m ... with 2 min jog recovery" as a `repeat` item with `repetitions: 6` and `steps: [work, rest]`.

Configuration surface (`WorkoutInterpreterOptions`):
```csharp
public sealed class WorkoutInterpreterOptions
{
    public string Model { get; init; } = "claude-sonnet-5"; // configurable; extraction task, not deep agentic reasoning
    public string? ApiKey { get; init; } // falls back to ANTHROPIC_API_KEY env var if null
    public int MaxTokens { get; init; } = 4096;
}
```
`claude-sonnet-5` is a reasonable default for this specific task — it's a single bounded extraction call producing at most a few hundred output tokens, not open-ended agentic work — but exposed as `Model` so the CLI/appsettings can override to `claude-opus-5` if a user wants stronger interpretation of unusually phrased workouts. Read `ApiKey` from `ANTHROPIC_API_KEY` unless explicitly overridden, never hardcode it, and never log it.

`ClaudeWorkoutInterpreter.InterpretAsync` flow: build `AnthropicClient`, call `client.Messages.Create(...)` (non-beta endpoint is sufficient — structured outputs via `OutputConfig.Format` doesn't require the beta namespace based on the tool-use.md example), extract the single `TextBlock` from `response.Content`, `JsonSerializer.Deserialize<WorkoutPlanDto>(...)`, then map `WorkoutPlanDto` (the wire-shape DTO with `type` discriminators) into the clean `WorkoutPlan`/`WorkoutItem` domain types shown in §2 (keep the DTO and domain model separate so the DTO can absorb schema quirks without polluting the mapper's input type).

## 4. Mapping layer (`FitWorkoutMapper`, `FitTargetEncoder`)

`FitWorkoutMapper.Map(WorkoutPlan) -> (WorkoutMesg, List<WorkoutStepMesg>)`:

- Walk `plan.Items` in order, tracking a running `messageIndex` counter (`ushort`, starts at 0).
- For a `SingleStepItem`: emit one `WorkoutStepMesg` via a `BuildStep(WorkoutStep, ushort messageIndex)` helper.
- For a `RepeatItem`: record `startIndex = messageIndex` before emitting its `Repeat.Steps` (each becomes its own `WorkoutStepMesg`), then after emitting all of them, append **one more** trailing `WorkoutStepMesg` exactly per the Cookbook's `CreateWorkoutStepRepeat` pattern:
  ```csharp
  var repeatMesg = new WorkoutStepMesg();
  repeatMesg.SetMessageIndex(messageIndex);
  repeatMesg.SetDurationType(WktStepDuration.RepeatUntilStepsCmplt);
  repeatMesg.SetDurationValue((uint)startIndex); // loops back to first step in the block
  repeatMesg.SetTargetType(WktStepTarget.Open);
  repeatMesg.SetTargetValue((uint)repeat.Repetitions);
  ```
  Use this exact raw-field pattern (not a hypothetical typed `SetRepeatSteps`) since it's Garmin's own verified working example and the field numbers (`DurationValue`=field 2, `TargetValue`=field 4) are unchanged in the current 21.217.0 profile.
- `BuildStep`:
  ```csharp
  var mesg = new WorkoutStepMesg();
  mesg.SetMessageIndex(messageIndex);
  if (step.Name is not null) mesg.SetWktStepName(step.Name);
  mesg.SetIntensity(MapIntensity(step.Kind)); // Warmup->Warmup, Work->Active or Interval, Rest->Rest, Cooldown->Cooldown
  MapDuration(mesg, step.Duration);
  MapTarget(mesg, step.Target);
  ```
- `MapDuration`: `TimeDuration` -> `SetDurationType(WktStepDuration.Time); SetDurationTime((float)seconds)`; `DistanceDuration` -> `SetDurationType(WktStepDuration.Distance); SetDurationDistance((float)meters)`; `OpenDuration` -> `SetDurationType(WktStepDuration.Open)`. Rely on the typed setters for the scale (confirmed correct per §0) — do not hand-multiply by 1000/100.
- `MapTarget` delegates the actual encoding to `FitTargetEncoder` to isolate the HR-convention risk area:
  ```csharp
  public static class FitTargetEncoder
  {
      public static void ApplyPaceRange(WorkoutStepMesg mesg, double fastMps, double slowMps)
      {
          mesg.SetTargetType(WktStepTarget.Speed);
          mesg.SetTargetValue(0);
          // Typed setters apply the true SDK scale (x1000) automatically — pass plain m/s.
          mesg.SetCustomTargetSpeedLow((float)slowMps);   // Low = slower end
          mesg.SetCustomTargetSpeedHigh((float)fastMps);  // High = faster end
      }

      public static void ApplyHeartRateZone(WorkoutStepMesg mesg, int zone)
      {
          mesg.SetTargetType(WktStepTarget.HeartRate);
          mesg.SetTargetHrZone((uint)zone); // 1-5, zone-based, no offset involved
      }

      public static void ApplyHeartRateRange(WorkoutStepMesg mesg, int lowBpm, int highBpm)
      {
          mesg.SetTargetType(WktStepTarget.HeartRate);
          mesg.SetTargetValue(0);
          // NOT an SDK scale/offset — this is a wire-format convention the watch
          // interprets: raw < 100 => %HR, raw >= 100 => absolute bpm (raw - 100).
          // We are always specifying absolute bpm here, so add 100 explicitly.
          mesg.SetCustomTargetHeartRateLow((uint)(lowBpm + 100));
          mesg.SetCustomTargetHeartRateHigh((uint)(highBpm + 100));
      }
  }
  ```
  This isolation is exactly what `FitTargetEncoderTests.cs` should pin down with explicit input->raw-byte-equivalent assertions (e.g. "135 bpm low -> `GetCustomTargetHeartRateLow()` returns 235").

  C# 14 note: since every `FitTargetEncoder` method's first parameter is the `WorkoutStepMesg` being mutated, this reads more naturally as C# 14 extension members (`extension(WorkoutStepMesg mesg) { public void ApplyPaceRange(...) { ... } }`) called as `mesg.ApplyPaceRange(fastMps, slowMps)`, rather than a static-class-with-static-methods. Either form is fine functionally — pick the extension-member form if targeting a cleaner call-site reads as worth it; keep the static-class form if extension members feel like unneeded ceremony for three small methods. Not a correctness issue either way.
- `WorkoutMesg` construction: `SetWktName(plan.Name)`, `SetSport(Sport.Running)`, `SetSubSport(SubSport.Invalid)` (generic; v1 doesn't need Track/Road/Trail distinction unless Claude infers it — could optionally map "on a track" mentions to `SubSport.Track`, but treat as a nice-to-have, not required for v1), `SetNumValidSteps((ushort)totalStepCount)`, optional `SetWktDescription(plan.Description)`.

## 5. FIT file writer (`FitWorkoutFileWriter`)

```csharp
public interface IFitWorkoutFileWriter
{
    void Write(WorkoutMesg workout, IReadOnlyList<WorkoutStepMesg> steps, Stream destination);
}

public sealed class FitWorkoutFileWriter : IFitWorkoutFileWriter
{
    public void Write(WorkoutMesg workout, IReadOnlyList<WorkoutStepMesg> steps, Stream destination)
    {
        var fileId = new FileIdMesg();
        fileId.SetType(Dynastream.Fit.File.Workout);
        fileId.SetManufacturer(Manufacturer.Development);
        fileId.SetProduct(0);
        fileId.SetTimeCreated(new Dynastream.Fit.DateTime(DateTime.UtcNow));
        fileId.SetSerialNumber((uint)new Random().Next());

        var encoder = new Encode(ProtocolVersion.V10);
        encoder.Open(destination);
        encoder.Write(fileId);
        encoder.Write(workout);
        foreach (var step in steps) encoder.Write(step);
        encoder.Close();
    }
}
```
Overload/wrap with a `Task WriteToFileAsync(WorkoutPlan plan, string path)` on the `WorkoutAgent` facade that opens a `FileStream`, calls the mapper then this writer, and closes the stream — mirroring the Cookbook's `CreateWorkout` sequencing exactly (FileId first, Workout second, steps in message-index order, then `Close()`).

## 6. CLI (`Fit.Agent.Cli`)

Minimal argument handling (no need for a heavy CLI-parsing package given the small surface):
- `fit-agent "<prompt text>" -o output.fit` — positional prompt argument (or `--stdin` flag to read from stdin if no positional arg given, per decision #2), `-o`/`--output` for destination path (default: sanitized workout name + `.fit` in the current directory).
- Optional `--model <id>` to override `WorkoutInterpreterOptions.Model` without touching env vars.
- Flow: parse args -> read `ANTHROPIC_API_KEY` (fail fast with a clear message if unset and no `--api-key` override) -> `await workoutAgent.CreateFitFileAsync(prompt, outputPath)` -> print success with the resolved path and step count, or catch and print a friendly error for: missing API key, Claude API errors (rate limit/timeout — via the SDK's typed exceptions), and JSON-schema/deserialization failures (surface Claude's raw response for debugging when `--verbose` is passed).
- Exit codes: 0 success, 1 usage error, 2 interpretation/mapping error.

## 7. Testing/verification strategy

1. **Unit tests, `FitTargetEncoderTests`** — call each `FitTargetEncoder` method with fixed inputs, then call the corresponding `Get...` accessor on the same `WorkoutStepMesg` instance (no encode/decode round-trip needed) and assert the *decoded* value matches the original input (e.g. `ApplyHeartRateRange(mesg, 135, 155)` then `mesg.GetCustomTargetHeartRateLow()` should be `135` again — the raw stored value is `235` but `Get` reverses scale/offset, so this test only proves the SDK's own get/set symmetry, not the +100 wire convention). Add a second explicit test that decodes at the **raw field level** if the SDK exposes it (check `Mesg.GetFieldValue`/`GetField` low-level accessors) to actually assert the raw byte value is 235 — this is the test that actually catches a regression in the manual +100 logic; the typed getter alone would not.
2. **Unit tests, `FitWorkoutMapperTests`** — build fixed `WorkoutPlan` fixtures (e.g. the "6x800m" prompt's expected parsed shape, and a simple linear warmup/tempo/cooldown plan) by hand (bypassing Claude entirely) and assert the resulting `List<WorkoutStepMesg>` has the right count, message indices, intensities, duration types/values, and — critically — that the trailing repeat-marker step has `DurationType == RepeatUntilStepsCmplt`, `DurationValue` equal to the first interval step's message index, and `TargetValue` equal to the repetition count.
3. **Integration test, round-trip via `Decode`** — encode a fixed `WorkoutPlan` fixture to an in-memory `MemoryStream`, then decode it back using `Decode` + `FitListener` (per `DecodeDemo.cs`) and assert: exactly one `FileIdMesg` with `Type == File.Workout`; exactly one `WorkoutMesg` with the expected name/sport/`NumValidSteps`; and the decoded `WorkoutStepMesgs` count and field values match what was written (catches any encoder/CRC-level corruption that a pure in-memory mapper test can't).
4. **Manual smoke test** (documented in README, not automated): run the CLI against a real Claude API call with the example prompt from the brief, inspect the resulting `.fit` file's size/existence, and optionally validate it opens correctly in a third-party FIT viewer or Garmin Connect's workout import before first real device use.
5. Explicitly **out of scope for automated tests**: no live Claude API calls in CI (non-deterministic, costs money) — `ClaudeWorkoutInterpreterTests`, if added, should mock `IWorkoutInterpreter`/use a canned JSON response rather than hitting the network.

## 8. Sequencing / build order

1. Scaffold solution + both csproj files + test project; add package references; verify `dotnet build` succeeds with just the FIT SDK reference (no Anthropic code yet) to de-risk the NuGet resolution/target-framework question first.
2. Build domain model (§2) with no dependencies — pure POCOs, quick to get right and unit-testable in isolation.
3. Build `FitTargetEncoder` + `FitWorkoutMapper` + `FitWorkoutFileWriter` against **hand-built** `WorkoutPlan` fixtures (no Claude yet) — this is the highest-risk, highest-payoff area given the scale/offset findings in §0, so get it correct and tested before touching the LLM integration.
4. Add the Decode-based round-trip integration test — proves the mapper+writer combination produces a genuinely valid file.
5. Build `IWorkoutInterpreter`/`ClaudeWorkoutInterpreter` + JSON schema, spike the structured-output call standalone first (a throwaway `Program.cs` or a manual test) to confirm the schema's `oneOf`/`$ref` nesting is actually accepted before wiring it into the DTO-to-domain-model conversion.
6. Wire the `WorkoutAgent` facade (interpreter -> mapper -> writer) and the CLI on top.
7. Write the README (usage, `ANTHROPIC_API_KEY` setup, manual watch-transfer instructions: USB mass storage into `GARMIN/NewFiles/` or Garmin Connect's "Import workout" feature).

### Critical Files for Implementation
- `src/Fit.Agent.Core/Mapping/FitWorkoutMapper.cs`
- `src/Fit.Agent.Core/Mapping/FitTargetEncoder.cs`
- `src/Fit.Agent.Core/Encoding/FitWorkoutFileWriter.cs`
- `src/Fit.Agent.Core/Interpretation/ClaudeWorkoutInterpreter.cs`
- `src/Fit.Agent.Core/Model/WorkoutPlan.cs`
- `/home/nick/Code/fit-csharp-sdk/Cookbook/WorkoutEncode/Program.cs` (reference only, do not copy)
- `/home/nick/Code/fit-csharp-sdk/Dynastream/Fit/Profile.cs` (reference only, source of truth for scale/offset)
