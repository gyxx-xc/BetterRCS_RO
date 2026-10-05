# BetterRCS_RO

Realism Overhaul compatibility patch for [BetterRCS](https://github.com/yobeefjerkyYBJ/BetterRCS_KSP).

BetterRCS swaps KSP's RCS thruster plumes for nicer Waterfall effects, but disables
itself when Realism Overhaul is detected. This patch bridges the two: it re-points RO's
Waterfall RCS config at the BetterRCS templates, so RO users get BetterRCS plumes while
keeping RO's RCS fuel/config system.

## Features

* Uses BetterRCS plume templates for every RO RCS thruster with an RO Waterfall config.
* Maps RO RCS fuel configs (`ModuleEngineConfigs`) to the matching BetterRCS plume variant.
* Has `lowslvac` / `lowsl` / `lowvac` / `base` / `hydrolox` / `methalox` six plume variants.

## Requirements

* [Realism Overhaul](https://github.com/KSP-RO/RealismOverhaul)
* [BetterRCS](https://github.com/yobeefjerkyYBJ/BetterRCS_KSP)

## Installation

[CKAN](https://github.com/KSP-CKAN/CKAN/wiki) install is recommended once available.

Manual: copy this repo's `GameData` folder into your Kerbal Space Program folder and
merge it, so you end up with:

```
Kerbal Space Program/
└── GameData/
    └── BetterRCS_RO/
        └── BetterRCS_RegexROSpecific.cfg
```

## Known Issues

* Position/rotation corrections are intentionally omitted — RO already places its
  RCS plumes well enough, so this patch only adjusts the scale.

## Contributing

Pull requests and issues are welcome. If you spot a plume that sits wrong, please
report it or fix it.

## Development

See [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) for a walkthrough of how the
patch, BetterRCS and RO's Waterfall pipeline work.

## Credits

* **BetterRCS** — [yobeefjerkyYBJ](https://github.com/yobeefjerkyYBJ/BetterRCS_KSP)
* **Realism Overhaul / RO Waterfall** — the KSP-RO team

## License

[MIT](LICENSE)
