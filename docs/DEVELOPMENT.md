# BetterRCS_RO — Development Notes

This document is written for people maintaining or extending the single config,
including myself.

file in this repo:

```
GameData/
└── BetterRCS_RO/
    ├── BetterRCS_RegexROSpecific.cfg   <-- all the logic lives here
    └── BetterRCS_RO.version
```

---

## 1. Upstream: how BetterRCS works

Files (`GameData/z_BetterRCS/`):

| File | Role |
| --- | --- |
| `Patches/BetterRCS_Regex.cfg` | The whole regex engine (see below). |
| `Templates/BetterRCS_*.cfg` | Six `EFFECTTEMPLATE` definitions. |
| `BetterRCS_Config.cfg` | Documentation for the opt-out tag. |

### 1.1 The templates

Each file defines one `EFFECTTEMPLATE` with a `templateName`:

| File | `templateName` | Meaning |
| --- | --- | --- |
| `BetterRCS_LVSP.cfg` | `BetterRCS_LowVacSLPressure` | Low pressure at both vacuum **and** sea level ("universal") |
| `BetterRCS_LSP.cfg`  | `BetterRCS_LowSLPressure`  | Low sea-level pressure |
| `BetterRCS_LVP.cfg`  | `BetterRCS_LowVacPressure` | Low vacuum pressure |
| `BetterRCS_Base.cfg` | `BetterRCS_Base`           | High-pressure / standard hypergolic |
| `BetterRCS_Hydrolox.cfg` | `BetterRCS_Hydrolox`   | LH2/LOX RCS |
| `BetterRCS_Methalox.cfg` | `BetterRCS_Methalox`   | CH4/LOX RCS |

A template is a big `EFFECTTEMPLATE` containing several `EFFECT` blocks
(`RCSBase`, `RCSMid`, `RCSEnd`, `Core`, `RCSJelly`, `RCSAtmoFuzz`, …). The
effects are driven by two controllers, referenced by **name** inside the
template:

* `atmosphereDepth` — blends vacuum vs. atmospheric look.
* `rcs` — drives brightness with RCS firing.

The template does **not** contain the controllers; whoever instantiates the
template (Watefall upstream, or RO here) must supply `CONTROLLER` nodes with
those two names.

### 1.2 The upstream allocation logic (non-RO installs)

BetterRCS's patch is guarded with `:NEEDS[Waterfall,!ROWaterfall]`, so on a
normal install it does the following:

1. **Initialization pass** — parts with `ModuleRCSFX` using `ElectricCharge`
   get a `BetterRCSDisabledTag` module so they are skipped.
2. **Regex pass** — reads the RCS `atmosphereCurve`, copies it to scratch
   `var_CurveISP*` nodes and rewrites keys with sentinel values to classify the
   thruster by ISP:
   * vacuum ISP `< 245` → tagged `Vac_LowPressure`
   * sea-level ISP `< 80` → tagged `SL_LowPressure`
3. **Scale pass** — adds an `atmosphereDepth` controller and squares up the
   plume scale.
4. **Allocation pass** — picks a template from the four ISP cases:

   | Vacuum low? | SL low? | Template |
   | --- | --- | --- |
   | no | no | `BetterRCS_Base` |
   | yes | no | `BetterRCS_LowVacPressure` |
   | no | yes | `BetterRCS_LowSLPressure` |
   | yes | yes | `BetterRCS_LowVacSLPressure` |

5. **Overwrite pass** — if the part burns `LqdHydrogen` → `BetterRCS_Hydrolox`;
   `LqdMethane` → `BetterRCS_Methalox`.
6. **Fluff pass** — appends "BetterRCS Configurated." to the description, and
   leaves a `BetterRCSModifiedTag` on parts it touched.

The matching is intentionally ISP-based because BetterRCS has to support
arbitrary mods. Applying that same regex engine under RO is exactly what
RO's own pipeline replaces.

---

## 2. Upstream: how the ROWaterfall pipeline works

RO does not store a finished `ModuleWaterfallFX` on each RCS part. Instead each
part carries a declarative `ROWaterfall { ... }` config node, and a series of
ModuleManager **factory passes** turns that node into one. The relevant files
live in `RealismOverhaul/Waterfall_Configs/`, mostly `_Processor/`.

A typical RO RCS config (e.g. `Squad/rcs.cfg`) just says:

```cfg
@PART[RCSBlock|...]:BEFORE[ROWaterfall]:NEEDS[Waterfall]
{
    %ROWaterfall
    {
        autoConfig = rcs
        transform  = RCSjet
        position   = ...
        rotation   = ...
        scale      = ...
    }
}
```

### 2.1 Pass order

| Pass (`:FOR[...]`) | What it does |
| --- | --- |
| `zROWaterfall_15_AutoConfig` | Expands `autoConfig = rcs`: sets `template`, `audio`, `defaultMainPlumeVariantName` and creates `MainPlumeVariant[monoprop]`/`[hypergolic]`; assigns `rowaterfallVariant` to `ModuleEngineConfigs` CONFIGs. |
| `zROWaterfall_80_MainPlumeVariants` | Builds one `ModuleB9PartSwitch` (`moduleID = rowaterfallMainPlumeSwitch`) with one `SUBTYPE` per `MainPlumeVariant`, copying the main module into each; links `ModuleEngineConfigs` CONFIGs to subtypes via `rowaterfallVariant`. |
| `zROWaterfall_90_Controllers` | Chooses the `controllerSet` from the template name and injects the matching `CONTROLLER` nodes. |

For RCS, `15_auto-config.cfg` sets:

```cfg
%template   = rowaterfall-rcs-cold-gas-1
%defaultMainPlumeVariantName = cold-gas
%MainPlumeVariant[monoprop]   { %template = rowaterfall-rcs-hydrazine-1 }
%MainPlumeVariant[hypergolic] { %template = rowaterfall-rcs-hypergolic-1 }
```

and maps `ModuleEngineConfigs` CONFIGs (`Hydrazine`, `NitrousOxide` →
`monoprop`; `MMH+MON3`, `MMH+MON10`, `*+NTO`, `Cavea-B` → `hypergolic`).

In `90_controllers.cfg`, templates matching `rowaterfall-rcs-*` get
`controllerSet = rcs`, which injects:

```cfg
CONTROLLER { name = atmosphereDepth linkedTo = atmosphere_density }
CONTROLLER { name = rcs             linkedTo = rcs                 thrusterTransformName = #$/ROWaterfall/transform$ }
```

Anything with no matching template gets the generic fallback
`controllerSet = atmo-thrust-basicRand`.

**Two consequences matter for us:**

1. Everything downstream of `ROWaterfall/template` (default plume, B9 subtypes,
   controllers) is generated from the `ROWaterfall` node. If you edit the node
   early enough, the generated config will pick up your changes.
2. After pass 80 the B9 subtypes are **copies** of the main module. Editing the
   finished `ModuleWaterfallFX` later only changes the default, not the copies
   inside each subtype.

---

## 3. How this repo works

Everything is in `BetterRCS_RegexROSpecific.cfg`. The file is deliberately
organized as four small patches.

### 3.1 Scale bump

```cfg
@PART[*]:HAS[@ROWaterfall:HAS[#autoConfig[rcs]],!MODULE[BetterRCSDisabledTag]]
      :AFTER[ROWaterfall]:NEEDS[Waterfall]
{ @ROWaterfall { @scale[*] *= 1.2 } }
```

RO's RCS configs already position the plume, so we don't need to do that again.
However, they are authored slightly larger than BetterRCS plumes expect, so the
`scale` on the source node is bumped 20 % before any generation pass reads
it. Because it runs before `zROWaterfall_18_LateInitialize`, the
`MainPlumeVariant`s inherit the adjusted value too.

`HAS[#autoConfig[rcs]]` restricts this to RCS thrusters (not engines), and
`!MODULE[BetterRCSDisabledTag]` honours the same opt-out mechanism BetterRCS
itself uses.

### 3.2 Inject and re-template the `ROWaterfall` node

```cfg
@PART[*]:HAS[@ROWaterfall:HAS[#autoConfig[rcs]],!MODULE[BetterRCSDisabledTag]]
      :AFTER[zROWaterfall_15_AutoConfig]:NEEDS[Waterfall]
{
    @ROWaterfall
    {
        %template = BetterRCS_LowVacSLPressure
        %defaultMainPlumeVariantName = lowvacsl

        %MainPlumeVariant[lowsl]     { %template = BetterRCS_LowSLPressure }
        %MainPlumeVariant[lowvac]    { %template = BetterRCS_LowVacPressure }
        %MainPlumeVariant[base]      { %template = BetterRCS_Base }
        %MainPlumeVariant[hydrolox]  { %template = BetterRCS_Hydrolox }
        %MainPlumeVariant[methalox]  { %template = BetterRCS_Methalox }
    }
    ...
}
```

This runs after RO's AutoConfig pass and overrides its result. Effects:

* The default plume becomes `BetterRCS_LowVacSLPressure`.
* Five extra plume variants are declared; pass 80 will build a B9 subtype for
  each, copying the BetterRCS-plumed module.
* `defaultMainPlumeVariantName` is renamed `cold-gas` → `lowvacsl`.

### 3.3 Re-map fuels to variants

```cfg
@MODULE[Module*EngineConfigs]:HAS[#type[ModuleRCS]]:NEEDS[B9PartSwitch]
{
    @CONFIG[Helium]         { %rowaterfallVariant = lowvac }
    @CONFIG[Hydrazine]      { %rowaterfallVariant = lowsl }
    @CONFIG[NitrousOxide]   { %rowaterfallVariant = lowsl }
    @CONFIG[MMH+MON3]       { %rowaterfallVariant = base }
    ...
    @CONFIG[CH4+O2]         { %rowaterfallVariant = methalox }
    @CONFIG[H2+O2]          { %rowaterfallVariant = hydrolox }
    // Nitrogen / HTP keep the default cold-gas -> lowvacsl
}
```

The MM part filter is the same as RO's.

Mapping summary:

| RO fuel CONFIG | `rowaterfallVariant` | BetterRCS template |
| --- | --- | --- |
| *(default: Nitrogen, HTP)* | `lowvacsl` | `BetterRCS_LowVacSLPressure` |
| `Helium` | `lowvac` | `BetterRCS_LowVacPressure` |
| `Hydrazine`, `NitrousOxide` | `lowsl` | `BetterRCS_LowSLPressure` |
| `MMH+MON3`, `MMH+MON10`, `*+NTO`, `RP-1+O2`, `Syntin+O2`, `Ethanol+O2`, `Cavea-B`, `ASCENT` | `base` | `BetterRCS_Base` |
| `CH4+O2` | `methalox` | `BetterRCS_Methalox` |
| `H2+O2` | `hydrolox` | `BetterRCS_Hydrolox` |

The fuel-to-variant mapping is informed by each fuel's ISP range.

### 3.4 Take over the controller set

RO picks a `controllerSet` by template name in pass 90, and would assign our
BetterRCS templates the generic `atmo-thrust-basicRand` set. Instead:

```cfg
// before RO's controller pass: declare our own set
@PART[*]:HAS[@ROWaterfall]:BEFORE[zROWaterfall_90_Controllers]:NEEDS[Waterfall]
{ @ROWaterfall:HAS[#template[BetterRCS_*]] { &controllerSet = betterrcs } }

// during RO's controller pass: inject the controllers
@PART:HAS[@ROWaterfall:HAS[#controllerSet[betterrcs]]]
      :FOR[zROWaterfall_90_Controllers]:NEEDS[Waterfall]
{
    @MODULE[ModuleWaterfallFX]:HAS[#__rowaterfall]
    {
        ATMOSPHEREDENSITYCONTROLLER { name = atmosphereDepth }
        RCSCONTROLLER
        {
            responseRateUp   = 1200
            responseRateDown = 20
            thrusterTransformName = #$/ROWaterfall/transform$
            name = rcs
        }
    }
}
```

Why not just reuse RO's stock `rcs` controller set? Because BetterRCS's plumes
want specific response rates. `responseRateDown = 20` makes the plume linger
after a puff, giving the "residual gas" look; the comment in the file calls
this out.

The `BEFORE` step is what prevents RO's generic fallback
(`@ROWaterfall:HAS[~controllerSet]`) from firing, since the node now has a
`controllerSet` value.

### 3.5 Full timeline (with this patch highlighted)

```
BEFORE[ROWaterfall]                 RO configs declare ROWaterfall{}
FOR[ROWaterfall]                    RO deletes old effects
AFTER[ROWaterfall]                  >>> BetterRCS_RO: scale *= 1.2
FOR[zROWaterfall_10_Initialize]     fill defaults
FOR[zROWaterfall_15_AutoConfig]     autoConfig=rcs expands
AFTER[zROWaterfall_15_AutoConfig]   >>> BetterRCS_RO: retemplate + re-map fuels
FOR[zROWaterfall_18_LateInitialize] inherit transform/pos/scale
FOR[zROWaterfall_20_MainEffect]     create ModuleWaterfallFX from template
FOR[zROWaterfall_80_MainPlumeVariants] build B9 subtypes (BetterRCS templates)
BEFORE[zROWaterfall_90_Controllers] >>> BetterRCS_RO: controllerSet = betterrcs
FOR[zROWaterfall_90_Controllers]    >>> BetterRCS_RO: inject custom controllers
FOR[zROWaterfall_99_Finalize]       description
FOR[zROWaterfall_Post]              (empty)
```

### 3.6 Opt-out

Any part carrying a `BetterRCSDisabledTag` module is left on RO's original
config. The tag is created by BetterRCS itself (for electric RCS) and can be
added by users; both patches in §3.1/§3.2 check for it.

---

## 4. Alternative approach — PR #12 (and why it is different)

PR #12 (`add RO compatibility`) is **not** the same design as this repo. It is
a ~30-line file dropped into BetterRCS's own folder:

```
GameData/z_BetterRCS/Patches/BetterRCS_RegexROSpecific.cfg
```

```cfg
@PART:HAS[@MODULE[ModuleRCS*]]:NEEDS[Waterfall,RealismOverhaul]:FOR[zz_BetterRCS]
{
    @MODULE[ModuleWaterfallFX]:HAS[@CONTROLLER[rcs]],*
    { @TEMPLATE { @templateName = BetterRCS_LowVacSLPressure } }

    @MODULE[ModuleB9PartSwitch]:HAS[#moduleID[rowaterfallMainPlumeSwitch]],*
    {
        @SUBTYPE[cold-gas],*  { @MODULE { @DATA { @TEMPLATE { @templateName = BetterRCS_LowVacSLPressure } } } }
        @SUBTYPE[monoprop],*  { @MODULE { @DATA { @TEMPLATE { @templateName = BetterRCS_LowVacPressure } } } }
        @SUBTYPE[hypergolic],*{ @MODULE { @DATA { @TEMPLATE { @templateName = BetterRCS_Base } } } }
    }

    MODULE { name = BetterRCSModifiedTag }
}
```

### 4.1 The two strategies

* **PR #12 — post-process the output.** Runs after RO has finished generating
  (`FOR[zz_BetterRCS]`, i.e. after the `zROWaterfall_*` passes), then directly
  overwrites `templateName` in the finished `ModuleWaterfallFX` and in each
  generated B9 `SUBTYPE`. It has no effect on RO's source node.
* **This repo — feed the generator.** Runs *inside* RO's pass sequence and
  edits the `ROWaterfall` source node (template + variants + fuel mapping) and
  the `controllerSet`, letting RO build the effects itself.

Because PR #12 edits output that pass 80 has already duplicated, it **must**
patch every B9 subtype by hand; this repo gets the subtypes for free by adding
`MainPlumeVariant`s before pass 80.

### 4.2 Concrete differences

| | PR #12 | This repo |
| --- | --- | --- |
| Location | inside `z_BetterRCS/Patches/` | standalone `GameData/BetterRCS_RO/` + CKAN/netkan |
| Part match | `@MODULE[ModuleRCS*]` (ModuleRCS *and* ModuleRCSFX) | `@ROWaterfall:HAS[#autoConfig[rcs]]` (only RO-generated RCS) |
| Hook point | post-process finished `ModuleWaterfallFX` | edit the `ROWaterfall` source node |
| Pass | `FOR[zz_BetterRCS]` (after RO) | `AFTER[ROWaterfall]`, `AFTER[zROWaterfall_15_AutoConfig]`, `BEFORE`/`FOR[zROWaterfall_90_Controllers]` |
| Variants | 3, mapped by **ISP range** (`cold-gas`→LVSL, `monoprop`→LV, `hypergolic`→Base) | 6 (default + `lowsl`/`lowvac`/`base`/`hydrolox`/`methalox`), mapped by **fuel CONFIG name** |
| B9 handling | edits each `SUBTYPE` explicitly | adds `MainPlumeVariant`s so pass 80 builds them |
| Controllers | leaves RO's `rcs` set as-is | replaces it with a custom set (`responseRateDown = 20`) |
| Scale/position | none (explicitly out of scope) | `scale *= 1.2` |
| Hydrolox/Methalox | omitted (out of scope) | included |
| RO module filter | `ModuleRCS*` covers legacy `ModuleRCS` | source-node based, so module class is irrelevant |
| Footprint | tiny, ships with BetterRCS | larger, separately installable/uninstallable |

The PR author's rationale (fuel variants mapped by ISP because RO's RCS scales
and `_Process` patches make per-part positioning hard) is sound for a minimal
upstream patch. This repo trades that minimalism for a broader, more accurate
mapping and custom controller feel, at the cost of tracking RO's internal pass
names.

### 4.3 Practical takeaway

If RO ever renames or reorders its `zROWaterfall_*` passes, **this repo breaks**
and the PR would not (it only depends on the final module shape). Keep the pass
names in §3.5 in sync with `RealismOverhaul/Waterfall_Configs/_Processor/`.
Conversely, post-processing cannot add new B9 subtypes or change controllers,
which is why this repo patches the source node.

---

## 5. Extending the patch

### Add a new fuel → variant mapping

1. Find the CONFIG name in `RealismOverhaul/Engine_Configs/RCS_Config.cfg`
   (or the relevant engine-config file).
2. Either point it at an existing `MainPlumeVariant` (add an `@CONFIG[...]`
   block in §3.3), or declare a new variant and its BetterRCS template in §3.2.
3. If you declare a new variant, RO's pass 80 will build the B9 subtype
   automatically — no need to touch B9PartSwitch manually.

### Add a whole new template

1. Ship an `EFFECTTEMPLATE` with the new `templateName`. It must
   expect controllers named `rcs` and `atmosphereDepth`.
2. Reference it from a `MainPlumeVariant` in §3.2.
3. Map at least one fuel CONFIG to that variant, or make it the default by
   changing `%template` / `%defaultMainPlumeVariantName`.

### Debugging

* `ModuleManager` logs each pass. Grep `KSP.log` for `BetterRCS` and for
  `zROWaterfall_90_Controllers` to confirm ordering.
* In-game, check a part's `ModuleWaterfallFX` in the PAW / config cache:
  `templateName` should start with `BetterRCS_`, and the module should have a
  controller named `rcs` with `responseRateDown = 20`.
* If a part still shows RO's plume, it either lacks
  `ROWaterfall:autoConfig[rcs]` or carries `BetterRCSDisabledTag`.

---

## 6. Known limitations

* Only parts that already have an RO `ROWaterfall` config can be converted;
  parts with Waterfall config are managed by BetterRCS;
  parts with no Waterfall config are untouched.
* Parts with multiple RCS modules may not get distinct plumes; ModuleManager
  cannot easily differentiate them from the source node alone.
* No position/rotation corrections are applied. If you find something wrong, pull requests and issues are welcome.
* Coupled to RO's internal pass names (§4.3).
